# Distance Matching 详解

> Distance Matching 做的事一句话就能说清：把「动画该播到哪一帧」从「过了多少时间」换成「走了多少距离」——给定一个距离，在曲线上反查出对应的时间，让动画里脚踩地的空间事件，和角色在游戏里真实走过的距离对齐，从而消灭滑步。

## 它解决的问题

动画是照着「固定速率 + 固定根运动」录制的：walk 循环里脚每 0.6 秒踩一下地，stop 动画里脚在某一帧踩死。但游戏里的角色不是匀速的——它起跑要加速、松摇杆要制动、落地要缓冲。用 DeltaTime 老老实实推进动画，就会出现：

- 起步时，胶囊已经开始往前滑，动画里的脚还在慢悠悠地抬；
- 停止时，胶囊已经刹住了，动画里那只脚还按原速往前多迈一步，脚在地上蹭；
- 落地时，不管离地多高，落地动画都按固定速度播，缓冲帧和实际触地时刻对不上。

本质是一个「空间—时间」错位：动画的时间轴是死的，角色在空间里的位移是活的。你要的是「脚踩地」这个空间事件恰好发生在「角色走到那个位置」的时刻。

## 它是一套工作流，不是一个 Pose 节点

先划清边界：Distance Matching 不是输入 Pose → 输出 Pose 的骨骼修正节点，而是 Animation Locomotion Library 插件里的一组**蓝图函数库函数**，靠 **Anim Node Function** 机制去改一个 **Sequence Evaluator** 节点的状态。三件套缺一不可：

| 组件 | 角色 |
|---|---|
| Distance Curve | 数据：记录「动画时间 t → 约定距离 d」 |
| Sequence Evaluator | 载体：一个「由外部显式时间驱动」的播放器 |
| Anim Node Function | 驱动：每帧把算好的时间写进 Evaluator |

之所以用 Sequence Evaluator 而不是 Sequence Player，源码注释写得很直白：它是「用显式时间输入去求值动画的某一点，而不是内部自行累加时间」，典型用途就是「播放位置代表时间以外的东西，比如跳跃高度」。Distance Matching 正是把它的 Explicit Time 当成「距离轴」来用。

## 核心思想：时间重参数化

普通播放：`t = t + Δt · PlayRate`，再 `Pose = Evaluate(t)`，时间是自变量。

Distance Matching 把自变量换成距离：`t = 反查(d)`。运行时由移动模型或测量得到一个目标距离 D，在 Distance Curve 上反解 `d(t) ≈ D` 的那个 t，写进 Sequence Evaluator 的 Explicit Time。曲线本身仍是「t → d」的正向映射，反查是运行时现做的。

源码里这个「反查」实现得相当朴素：**二分查找 + 线性插值**——先在递增的曲线键值里二分定位 D 落在哪两个键之间，再按比例在时间轴上插值。它默认了两个前提：键值单调递增、每个值唯一对应一个时间（否则「一个距离对应多帧」就没法唯一反查）。

### 两种用法：绝对反查 vs 相对前进

库提供了两条路，对应两类场景：

- **`DistanceMatchToTarget`**：绝对定位。输入「到目标的剩余距离」，直接反查出对应时间整帧跳过去。用在停止、转向这种「目标明确」的时刻。
- **`AdvanceTimeByDistanceMatching`**：相对前进。输入「这一帧走了多少距离」，从当前时间沿曲线一路累计，累计够这个距离就停。用在起步这种「还没到目标，只要跟上位移」的时刻。

值得注意的细节：`AdvanceTimeByDistanceMatching` 内部其实会把「走够这段距离需要的时间」换算回一个 **effective play rate**，再按这个速率前进，并可以 clamp。也就是说，即便走的是「前进」这条路，最终落地也是「算一个能满足位移的播放速率」——这就引出了它和 Play Rate 的关系。

## 和 Play Rate 的本质区别

Play Rate 是对整条时间轴做**线性缩放**：乘以 1.25，所有帧一起快 25%，脚抬得快、落下得也快，但「脚落地」这个事件在距离轴上的位置没变——它还是发生在动画录制的那个位移处，而角色此刻未必真的走了那么远。

Distance Matching 不做均匀缩放，它**直接挑「对应这个距离的那一帧」**：角色离停点还剩 40cm，就播「还剩 40cm 时脚该踩下去」的那一帧；还剩 5cm，就播最后那几帧。脚的空间事件被强行钉在游戏距离上。

但库自己的头注释又补了一句诚实的话：Distance Matching 本质上仍是「改变播放速率来让脚不滑」，所以通常要 clamp 这个速率，避免动画播得过慢或过快，差值交给 Stride Warping 之类的姿势修正去补。这条正好解释了 `PlayRateClamp`（默认 0.75–1.25）存在的理由：clamp 一开，极端情况下会重新引入一点点脚滑，这是「别让动画慢到像慢动作」和「绝对不滑」之间的刻意取舍。

对**循环动画**（走路、跑步）根本不需要 Distance Matching——它们速度恒定，均匀缩放就够了。库里的 `SetPlayrateToMatchSpeed` 就是干这个的：`PlayRate = 目标速度 / 动画自带速度`。距离匹配是给**过渡/一次性动画**（起步、停止、转向、落地）用的，循环态用 Play Rate 对齐即可。

## Distance Curve：方向、零点、单位

曲线是 `UDistanceCurveModifier` 从 root motion 烘焙出来的。它的逻辑要点：

1. 先找**停止/转向点**：要么指定「片尾即停止」（`bStopAtEnd`），要么以 1/120s 的高分辨率扫一遍，找 root motion 速度低于 `StopSpeedThreshold` 的那一帧，当作「最小速度点」。
2. 再以固定采样率（默认 30Hz）逐点算「该时刻相对停止点的位移」，按指定轴取模长（默认 XY 平面），写入曲线。
3. **符号约定**是关键：停止点之前是**负值**（表示「到停止点还剩多少距离」），之后是**正值**（表示「从停止点出发已走多少距离」）。

这也是为什么 `DistanceMatchToTarget` 里要先把输入取负号——因为「到目标的剩余距离」在曲线里约定成负数存储，运行时输入是正数，就得用 `-DistanceToTarget` 对齐。曲线的方向、零点（谁是 0）、单位（水平位移还是垂直高度、XY 模长还是单轴）必须和你运行时的输入完全同义，否则反查出来的帧就是错的。落地场景就是把这条约定换到 Z 轴：曲线记录「离地高度」，运行时输入实际离地高度，反查出对应的缓冲帧。

编辑器源码里也留了 TODO：这套烘焙对「只有一个停止/转向点」的简单 clip 可靠，多停止点、靠方向变化检测 pivot 这些都没做——复杂 clip 要手工检查甚至手调曲线。

## 和谁配合：距离从哪来

Distance Matching 自己**不算距离，也不选动画**，只负责「给距离，还你时间」。距离由外部喂进来：

- **停止**：`PredictGroundMovementStopLocation` 用 UCharacterMovementComponent 的制动参数（制动摩擦、地面摩擦、步行减速度）预测停点，返回局部空间向量，其**长度就是到停点的距离**，喂给 `DistanceMatchToTarget`。
- **转向（Pivot）**：`PredictGroundMovementPivotLocation` 预测「速度反向的那一刻」的位置，同样取长度。
- **起步**：已经前进的距离（速度积分或轨迹），喂给 `AdvanceTimeByDistanceMatching`。
- **落地**：离地高度（trace 或位置差），配合 Z 轴高度曲线反查。

四个场景各用一个「移动模型/测量」得到 D，再走同一套反查。链路是 `预测/测量 → 反查 Distance Curve → 写 Explicit Time`，不越界。

## 常见问题

- **写了显式时间但动画不动**：`SetExplicitTime` 会返回失败并提示「Set it as Always Dynamic」——Explicit Time 引脚被常量折叠了。把引脚暴露成动态输入，Anim Node Function 才写得进去。
- **反查结果明显错帧**：几乎都是曲线方向/零点/单位与输入不一致。先对一对：你输入的是「剩余距离」还是「已走距离」？曲线里负数是什么含义？烘焙用的是 XY 还是 Z？
- **从 jog 直接接停止，脚相位乱跳**：`DistanceMatchToTarget` 完全由剩余距离决定时间，**不尊重上一段动画的相位**（源码注释原话）。这是特性不是 bug，想平滑接相位得另做处理（如接受相位跳变，或先用惯性混合切序列）。
- **曲线查不到 / 动画覆盖距离不足**：库有防护——曲线范围接近 0 会直接告警并放弃，避免死循环。检查动画是否真的开了 root motion、曲线是否烘出来了。
- **Notifies 不触发**：Sequence Evaluator 明确不触发序列里的 Notify（脚步事件等要靠别的方式补）。
- **用了 Distance Matching 脚还滑**：多半是 `PlayRateClamp` 在起作用，或者该段动画根本没被距离驱动（循环态忘了用 Play Rate 对齐）。记得配合 Stride Warping 补差值。

## 参考资料

- [Distance Matching（官方文档）](https://dev.epicgames.com/documentation/en-us/unreal-engine/distance-matching-in-unreal-engine)
- [Animation Locomotion Library API](https://dev.epicgames.com/documentation/en-us/unreal-engine/API/Plugins/AnimationLocomotionLibrary)
- [Game Animation Sample Project](https://dev.epicgames.com/documentation/en-us/unreal-engine/game-animation-sample-project-in-unreal-engine)
- 引擎源码（UE 5.8）：
  - `Engine/Plugins/Animation/AnimationLocomotionLibrary/Source/Runtime/`（`AnimDistanceMatchingLibrary.h/.cpp`、`AnimCharacterMovementLibrary.h/.cpp`）
  - `Engine/Plugins/Animation/AnimationLocomotionLibrary/Source/Editor/Private/DistanceCurveModifier.cpp`（曲线烘焙）
  - `Engine/Source/Runtime/AnimGraphRuntime/Public/AnimNodes/AnimNode_SequenceEvaluator.h`（Explicit Time 语义）
  - `Engine/Source/Runtime/Engine/Classes/Animation/AnimCurveCompressionCodec_UniformIndexable.h`（`FAnimCurveBufferAccess` 的按索引读取，曲线需 Uniform Indexable 压缩）
