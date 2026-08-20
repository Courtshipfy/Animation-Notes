# Blend Stack 详解

> Blend Stack 做的事一句话就能说清：它不再用「固定 A、B 两路」去做过渡，而是维护一叠正在播放的候选动画，最新来的放最顶上、权重从它开始往下一层层递减，老的自然淡出、淘汰；当 Motion Matching 每帧都在改选动画时，这叠「栈」能持续吸收新候选而不退化成一次性的两路混合。

## 它解决的问题

Motion Matching（或其他每帧改选动画的选择系统）有一个传统混合结构扛不住的特性：**候选动画可能每帧都在变**。这一帧选跑步 A，下一帧可能换成跑步 B，再下一帧又换成转向 C。用「A→B 两路 blend、播完再切」的老思路，候选一变就得重新搭过渡，频繁重定向下要么 Pop、要么混合状态爆炸。

Blend Stack 就是为「持续重定向的姿势流」准备的：它不把混合压成两个状态，而是保留一叠**同时存在、各自有淡入权重的候选**。新候选来了就压到栈顶淡入，旧候选的权重被自然挤掉、淡出、最后被剪枝。因为混合上下文一直在栈里延续，它天然能承受「每帧换一个候选」的节奏。

它明确**不做**三件事，理解边界才不会误用：

- **不判断候选对不对**：它只负责把给到的候选平滑地接起来，选错动画是上游（Motion Matching / Chooser）的事，它照样会把错的姿势「平滑地」播出来。
- **不是姿势修正器**：它不掰腿、不算 IK。它是 `FAnimNode_AssetPlayerBase` 的子孙——它自己**拥有并播放**那些候选动画（内嵌 Sequence Player / Blend Space Player），而不是在别人给的姿势上做加工。
- **不是必须品**：只有少量固定状态跳转时，普通状态机 + 两路 Blend 更直白，没必要为了它重写稳定状态机。

## 核心思想：一叠「栈」，权重从顶向下递减

节点内部维护一个 `AnimPlayers` 数组，**索引 0 是最新插入的（主玩家），索引越大越旧**。每帧求值时：

1. 最新玩家的「淡入权重」由它自己的 blend-in 进度决定（`BlendInPercentage → 权重曲线`）。
2. 剩下的权重（`1 − 最新玩家权重`）再分给下一个玩家，依次乘下去——形成一条「最新占大头、越旧越小」的递减权重链。
3. 权重被挤到 0 的玩家直接剪掉（`PopLastAnimPlayer`），不再求值。

用一句话概括：**它不是「两路各 50%」，而是「最新一个淡入、其余按剩余权重瓜分、最老的被自然挤死」**。这也是它和普通 Two-Way Blend 最本质的区别——普通 blend 每次过渡都要重新定义「谁是 A 谁是 B」，Blend Stack 永远在一条连续递减的权重链上滑。

每帧还有一个关键动作：用 `Context.FractionalWeight(权重)` 给每个玩家构造自己的更新上下文。这样嵌套在里面的子图（见下文「每个候选的专属子图」）能看到自己被缩小的权重，Notifies、Root Motion 等也按权重正确折算。

## 关键流程：新候选来了怎么办

当上游请求一段新动画时，节点调 `BlendTo`。它不是无脑压栈，而是先做几个判断：

1. **栈空 → 直接进**：没有历史，`BlendTime` 置 0，第一段动画直接满权重进入，不做过渡。
2. **上一个还没淡入完 → 顶掉它**：如果栈顶玩家还没完成淡入、且淡入时间小于 `MaxBlendInTimeToOverrideAnimation`（默认 0），就用新请求**替换**栈顶，而不是再压一层。这是给「Motion Matching 极快改选」设计的合并：候选还没起来就被换掉，就别再堆一层了。
3. **正常情况 → 压栈**：插入一个新的玩家到索引 0，新玩家带自己的 `BlendTime`、`BlendProfile`、镜像、同步组等信息开始淡入。

新玩家压栈时，会初始化一个内嵌的 Sequence Player 或 Blend Space Player（按资产类型），支持循环、播放速率、镜像、同步组。**Montage 不被支持**（初始化时直接报 unsupported），因为 Blend Stack 面对的是持续姿势流，不是 Montage 的槽位语义。

### 溢出怎么办：MaxActiveBlends 与「烤成静态姿势」

同时存在的活跃玩家是有限的，上限由 `MaxActiveBlends`（默认 4）控制。一旦压进来的玩家超过这个数，求值阶段会做一件事：

- 把「从第 MaxActiveBlends 层往下」的那些老玩家，**逐个混进累计姿势**，然后弹出；
- 若 `bStoreBlendedPose`（默认 true）开启，这个累计出来的姿势被**烤成一份静态快照**存进最后一个玩家——它从此不再播动画，只是「一坨已经混好的姿势」，作为背景继续存在；
- 若 `bStoreBlendedPose` 关闭，为了省内存，超出的玩家直接丢弃，代价是可能 Pop（源码会在丢掉的玩家权重还明显时打 Warning）。

这个「溢出烘焙」是 Blend Stack 能用有限内存承受无限次重定向的关键：**活跃的永远只有 MaxActiveBlends 个真动画，其余全部坍缩成一份静态姿势**。

## 每个候选的专属子图：Blend Stack Graph

这是 Game Animation Sample 里最容易让人困惑、也最关键的一块。Blend Stack 节点本身可以挂一个「采样图（Sample Graph / Blend Stack Graph）」：节点维护一个子图池（`PerSampleGraphPoseLinks`），每个活跃玩家被分配一个子图槽位（轮询复用），**这个玩家的姿势先流经自己那条子图，再回到栈里参与混合**。

子图内部用 `FAnimNode_BlendStackInput` 作为入口——它不是一个「外部输入」，而是「当前被分配到本槽位的那一个玩家」的替身：子图从这里拿到该玩家正在播的动画，往后面接任何想做的处理。

这正是 GASP 把 **Orientation Warping 和 Steering 放在 Blend Stack 内层**的实现方式：每个候选在进入混合前，都先经过同一套方向/轨迹修正。好处有两个：

- **修正作用于每个候选**，而不是只作用于混合后的结果——混合出的姿势每一路都已经被掰到正确的方向；
- **不会内外层重复修正**——如果把这套 Warping 又放在主 AnimGraph 外层，方向/根运动就被修了两次。

一句话：**Blend Stack 负责「接住不断变化的候选」，子图负责「让每个候选先被修正一遍」，两者合起来就是 Motion Matching 的混合核心。**

## 交给惯性化的那一路：bUseInertialBlend

Blend Stack 默认用普通交叉淡化（带 BlendProfile 时按骨骼权重）。但它也提供 `bUseInertialBlend`：开启后，`BlendTo` 不再自己做线性 blend，而是向下游发一条 **Inertialization Request**（可带 `InertialBlendNodeTag` 指定命中某个 Inertialization / Dead Blending 节点），然后把 `BlendTime` 置 0。

这实际上是把「过渡」这件活外包给了 [Inertialization](./Inertialization.md) / [Dead Blending](./Dead-Blending.md)：Blend Stack 只管「切换到新姿势」，接缝的平滑交给惯性化去外推。用这条路径时，栈里的权重变化和普通 blend 手感会不同——残差是惯性衰减，而不是线性交叉。

## 几个容易忽略的旋钮

- **`BlendspaceUpdateMode`**（`InitialOnly` / `UpdateActiveOnly` / `UpdateAll`）：当候选是 Blend Space 时，控制 XY 采样参数多久更新一次。默认只在淡入时更新一次，避免正在淡出的旧 Blend Space 每帧追着新输入跑。
- **`PlayerDepthBlendInTimeMultiplier`**：越深的玩家淡入计时器按 `multiplier^depth` 加速/减速。它影响「老玩家是慢吞吞还是快速被挤掉」。
- **`ActivationDelayTime` / 延迟玩家**：请求可以带一个激活延迟，延迟期内的玩家被视为「未激活、未开始淡入」，此时若来了新请求，这个被推迟的玩家会被直接丢弃——同一时刻只允许一个推迟玩家。
- **`bResetOnBecomingRelevant`**（默认 true）：节点被缓存跳过、又回到相关路径时，重置整个栈，避免把失联期间的老混合带回来。
- **`bShouldFilterNotifies` + `NotifyRecencyTimeOut`**：多个堆叠玩家可能在短时间内重复触发同一条 Notify（脚步、音效），这个开关按「近期触发过就过滤」去重。
- **StitchDatabase（实验）**：可选地配一个 PoseSearch 数据库，`BlendTo` 时先在数据库里搜一段「缝合动画」桥接当前姿势与目标姿势，再通过缝合动画过渡过去。超出 `StitchBlendMaxCost` 或数据库缺失就退回普通 blend。实验性，生产慎用。

## 和谁配合、放在哪

- **上游：Motion Matching / Chooser Player**。它消费的是「资产 + 时间 + 播放速率 + 混合时间」的请求流，不自己选动画。Motion Matching 每帧喂新候选，Blend Stack 负责接住。
- **内层：Orientation Warping + Steering**。放子图（Sample Graph）里，让每个候选先被修正。详见上节。
- **下游或外包：Inertialization / Dead Blending**。`bUseInertialBlend` 把接缝外包出去。
- **内部复用**：Chooser Player（`FAnimNode_ChooserPlayer`）内部就是一个 Blend Stack，用来播 Chooser 选出的资产——这解释了为什么 Chooser 选完资产后能平滑过渡。

## 常见问题

- **频繁改选时 Pop**：先看 `MaxBlendInTimeToOverrideAnimation` 是否为 0——为 0 时「上一个还没淡入就压新层」，若你的上游极快改选，候选会堆得很快。适当调大它让相邻请求合并，比堆 `MaxActiveBlends` 更对症。
- **混着混着「冻住」了**：这是溢出烘焙在起作用——超出 `MaxActiveBlends` 的老玩家被烤成静态姿势。静态姿势不再随动画推进，若它权重还不小，会看起来「卡住」。调大 `MaxActiveBlends`，或缩短 `BlendTime` 让老玩家更快淡出。
- **内存吃紧 / 想省内存**：关掉 `bStoreBlendedPose` 能省下烘焙姿势的内存，但代价是超出上限的玩家被硬丢、可能 Pop。取舍点是「能不能接受偶尔的接缝跳变」。
- **给的是 Montage，不播**：Blend Stack 不支持 Montage，初始化直接报 unsupported。Montage 走 Slot / 状态机，不是这套持续姿势流。
- **用了 Warping 却感觉被修了两次**：检查是不是既在子图里放了 Orientation Warping / Steering，又在主 AnimGraph 外层重复放了一层。修正在子图内层做一次即可。
- **Notifies 重复触发**：多个玩家同时逼近同一事件时可能重复开火。开 `bShouldFilterNotifies` 并配 `NotifyRecencyTimeOut`。

## 参考资料

- [Blend Stack（官方文档）](https://dev.epicgames.com/documentation/en-us/unreal-engine/blend-stack-in-unreal-engine)
- [Game Animation Sample Project](https://dev.epicgames.com/documentation/en-us/unreal-engine/game-animation-sample-project-in-unreal-engine)（Blend Stack Graph 内层放置 Orientation Warping 与 Steering 的官方示例）
- 引擎源码（UE 5.8）：`Engine/Plugins/Animation/BlendStack/Source/Runtime/`
  - `Public/BlendStack/AnimNode_BlendStack.h`（`FAnimNode_BlendStack_Standalone` 与 `FAnimNode_BlendStack`，栈与请求字段）
  - `Private/AnimNode_BlendStack.cpp`（`InternalBlendTo` 的压栈/替换逻辑、`UpdateAssetPlayer` 的权重递减、`Evaluate_AnyThread` 的溢出烘焙、`RequestInertialBlend`）
  - `Public/BlendStack/AnimNode_BlendStackInput.h`（采样图入口节点）
