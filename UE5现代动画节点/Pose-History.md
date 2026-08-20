# Pose History 详解

> Pose History 做的事一句话就能说清：给 Motion Matching 补上「过去」这个维度——它把每一帧的骨骼姿势连同时间戳滚进一段窗口，让查询在比较候选动画时，不仅知道「这帧腿在哪」，还知道「刚才这帧是怎么动的」。它和 Cached Pose 的本质区别也在这里：Cached Pose 复用当前帧的同一份计算结果，Pose History 提供的是跨帧的运动历史。

## 它解决的问题

Motion Matching 的核心动作，是把数据库里的候选姿势和「角色当前的状态」做比对。但「当前状态」从来不只是这一帧的静态姿势：

- 要匹配「速度」，就得知道某根骨头**上一刻**在哪、**这一刻**在哪，才能算出变化的方向和大小；
- 要匹配「未来 0.5 秒脚的位置」，就得有能力回答「带时间偏移的采样：这根骨头在 t 时刻在哪」。

如果只把这一帧的姿势塞给查询，速度、位移这些「运动量」根本无从谈起——它们本质是两帧之差。Pose History 就是那个能按时间回答「这根骨头在 t 时刻在哪」的组件：它持续把每帧姿势压进一个带时间戳的滚动缓冲，供查询按任意时间点采样。

它明确**不做**的事，理解了边界才不会误用：

- **不参与选动画**：它只是数据的提供者，真正做搜索的是 Motion Matching 节点。
- **不修改姿势**：它是旁路采样，输出姿势原样透传，只是把自己的指针挂到输出属性上。
- **不自动知道要采什么**：采哪些骨骼、采多密，由你通过 `CollectedBones` / `SamplingInterval` 配置；Schema 关心什么就采什么。

## 核心思想：一段带时间戳的姿势快照队列

Pose History 内部是一段环形缓冲，最多存 `PoseCount` 条记录。每条记录 = 一组骨骼的**组件空间**变换（旋转 + 平移，可选缩放）+ 一组曲线值 + 一个时间戳。时间约定是：**0 = 当前帧，负数 = 过去，正数 = 未来**（未来样本来自轨迹预测或根骨恢复）。

每一帧它对缓冲做三件事（对应源码 `EvaluateComponentSpace` 的主干）：

1. **老化**：所有旧样本的时间戳减掉 `DeltaTime`，整段窗口向前滑一格；
2. **取舍**：最新一条永远被当前姿势覆盖成 `0`；旧样本按 `SamplingInterval` 的节拍把最旧的一条回收、腾出槽位（`SamplingInterval = 0` 表示每帧都收一条）；
3. **写入**：把本帧姿势写进最新条。

于是 `PoseCount` 决定「最多记住几条、往回看多远」，`SamplingInterval` 决定「采样多密」——两者合起来约定这段历史的时间窗口：样本越多、越密，区分力越强，但每帧写入和内存成本越高。

查询走按时间取样：`GetTransformAtTime(time, ...)` 在缓冲里二分找到包围 `time` 的两条记录，插值出结果，必要时还能外推。速度这类量就是靠两次取样做差分：

```text
速度 ≈ ( 位置(time) − 位置(time − Δt) ) / Δt
```

这正是查询上下文里 `GetSampleVelocity` 的实际做法：取「当前样本」和「Δt 之前的样本」两次位置相减，再除以步长。

## 和 Cached Pose 的本质区别

两者名字里都有「缓存」，目的却完全不同，混用是新手最常见的坑：

| | Cached Pose | Pose History |
|---|---|---|
| 复用什么 | 当前帧的**同一份**本地空间姿势 | 过去多帧的组件空间姿势 |
| 时间维度 | 无，就是「这一帧」 | 有，按时间戳索引，可回看过去 |
| 用途 | 把一段计算复用给图里多个下游节点，省重复求值 | 给查询提供「刚才怎么动」的时间序列 |
| 典型读法 | 图里另一个节点直接当输入接 | Motion Matching 通过图消息取指针，按时间查询 |

一句话：Cached Pose 解决「同一帧算一遍」，Pose History 解决「记住刚才那几帧」。

## 关键流程：写入、暴露、消费

1. **建实例**：节点初始化时创建一个跨帧存活的共享 `FPoseHistory` 对象（`PoseHistoryPtr`），它不属于任何单帧。
2. **初始化**：按 `PoseCount` / `SamplingInterval` 配好缓冲，先用参考姿势填一次 `0` 时刻，避免首帧查询扑空。
3. **更新（Update）**：写入查询轨迹（`TransformTrajectory` 引脚，或实验性的 `bGenerateTrajectory` 自生成），同步「是否 relevant」的计数器。
4. **求值（Evaluate）**：算 `bNeedsReset`，从 `CollectedBones` 解出需要采样的骨骼索引，调用历史对象的求值函数完成「老化 + 覆盖最新」，再把历史指针塞进一个自定义属性供下游访问。
5. **发布**：节点在 Update 时用 `FPoseHistoryProvider` 图消息把自身挂到动画图上下文。
6. **消费**：下游 Motion Matching 节点在更新资产时 `GetMessage<FPoseHistoryProvider>()` 取回历史指针——拿不到就直接报 `missing IPoseHistory`。之后构建查询时，Schema 里的 Pose / Velocity / Position / Heading 等通道各按自己的 `SampleTimeOffset` 调 `GetTransformAtTime` / `GetCurveValueAtTime` 采样。

## 采样多少：区分力与成本

采什么、采多密，是这段历史的「分辨率」，两端都要权衡：

- **骨骼范围**：`CollectedBones` 空着时，节点会把**整副骨架所有骨骼**都收进每条样本（源码里映射表为空即「收全部」），省事但每条样本的内存和拷贝开销很大。实际做法是按 Schema 关心的骨骼列出来；根骨无论列没列都会被强制收集（保证至少有一个可用的参考变换）。
- **时间密度**：`PoseCount` 越大、`SamplingInterval` 越小，历史越远越密，快速动作的区分力越强，但缓冲越大、每帧写入越多。
- **曲线**：`CollectedCurves` 里列出的曲线才会存进历史。Schema 用了某条曲线却没列，查询时会打警告 `Couldn't find Curve ... Consider adding it to the Pose History!`；骨骼同理会提示 `Couldn't find BoneIndexType ... Consider adding it to the Pose History!`。

这些运行时警告是调配置最直接的线索，比猜强得多。

## 必须持续更新，不能「进状态只记一次」

Pose History 的全部价值建立在「每帧求值」上：样本要每帧老化、每帧补最新，窗口才滚得起来。这带来两个由源码行为直接印证的坑：

- **放在不走的支路上 = 读到过期姿势**。只有求值函数会老化和写入样本。如果节点挂在只在进入状态时求值一次的分支（或布尔混合的永假侧），历史就冻结在最后一次写入的状态上，查询会一直拿到那份过期姿势。
- **重新 relevant 会被清空**。`bResetOnBecomingRelevant`（默认开）配合一个更新计数器检测「上一帧没更新、这一帧又更新了」的情况，一旦发生就重置历史，防止把失联期间的老数据当成当前状态。代价是：从休眠切回活跃的那一瞬，历史是空的，需要几帧重新攒满。

正确摆法是：把 Pose History 放在和 Motion Matching **同一条常驻路径**上、且排在它上游，两者每帧一起走。

## 它不碰什么、和谁配合

- **只管收集，不管搜索**：搜索、混合、惯性化是 Motion Matching 节点的职责，Pose History 只负责把历史通过图消息递给它。
- **轨迹也住在它这里**：查询用的轨迹（`TransformTrajectory`）挂在 Pose History 节点上，`GetTransformAtTime` 求世界空间时用轨迹样本把组件空间变换转到世界空间。这也是它叫「History Collector」的原因——它同时是历史与轨迹的容器。
- **两个变体对应两种摆放位置**：普通版接本地空间 `FPoseLink`，组件空间版接 `FComponentSpacePoseLink`。想让历史包含 Leg IK 等骨骼控制节点修改后的结果，就用组件空间版并放在那些节点之后。
- **实验性的根骨恢复**：`RootBoneRecoveryTime` 系列参数用于 Offset Root Bone 场景，把漂移的根骨逐渐拉回参考姿势（源码明确标注 Experimental，可能随时删改，不建议生产使用）。

## 常见问题

- **Motion Matching 报 `missing IPoseHistory`**：历史节点没被求值，或不在同一 relevant 路径上，图消息传不过来。先确认两个节点在同一常驻分支、且 Pose History 在 MM 上游。
- **查询一直命中「过期姿势」**：节点只在进入状态时记了一次，之后没再走求值。把它挪到常驻路径，每帧更新。
- **日志刷 `Consider adding it to the Pose History!`**：Schema 用到的骨骼/曲线没在 `CollectedBones` / `CollectedCurves` 里列出来。按警告补齐即可。
- **历史表现「钝」、细节匹配不出来**：`PoseCount` 太小或 `SamplingInterval` 太大，窗口/密度不够。反之性能吃紧就先砍 `CollectedBones` 里不用的骨骼，再考虑调大采样间隔。
- **缩放读出来是单位缩放**：`bStoreScales` 默认关，缩放被假定为 1。只有 Schema 真的匹配缩放时才需要开。
- **切状态瞬间匹配跳变**：多半是 `bResetOnBecomingRelevant` 在重置历史——重新 relevant 的头几帧历史为空，属预期；若不想清零就把它关掉（但要接受可能吃到失联期的老数据）。

## 参考资料

- [Motion Matching in Unreal Engine](https://dev.epicgames.com/documentation/unreal-engine/motion-matching-in-unreal-engine)
- [Game Animation Sample Project](https://dev.epicgames.com/documentation/en-us/unreal-engine/game-animation-sample-project-in-unreal-engine)
- 引擎源码（UE 5.8）：`Engine/Plugins/Animation/PoseSearch/Source/Runtime/`
  - `Public/PoseSearch/AnimNode_PoseSearchHistoryCollector.h`（节点字段与两个变体）
  - `Private/AnimNode_PoseSearchHistoryCollector.cpp`（写入、复位、图消息发布）
  - `Public/PoseSearch/PoseSearchHistory.h` / `Private/PoseSearchHistory.cpp`（环形缓冲与按时间取样）
  - `Private/PoseSearchContext.cpp`（通道按 `SampleTimeOffset` 查询、速度差分）
