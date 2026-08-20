# Dead Blending 详解

> Dead Blending 干的还是 Inertialization 那类事：在动画切换的瞬间把「跳跃」吸收掉。但它走的是另一条路——不往目标姿势上叠一个逐渐衰减的「差量」，而是把被切走的旧动画顺着它自己的运动速度「往前外推」，再把这段外推出来的"幽灵姿势"和新动画做一次普通的交叉淡化。

## 它解决的问题

两个动画硬切，必然在切换帧出现不连续：旧动画的手臂摆在这里，新动画的手臂摆在别处。Inertialization 的思路是，切换瞬间量出「旧姿势相对新姿势的偏移」，然后让这个偏移在几百毫秒内平滑衰减到零——姿势被拽向新动画，但过程没有 Pop。

Dead Blending 解决的是同一个问题，但它对"切换之后旧动画该去哪"给出了不同答案。Inertialization 让旧姿势**被拽向新姿势**；Dead Blending 让旧动画**继续按惯性往前跑一小段**，同时慢慢把控制权交给新动画。这两种手感不一样：前者是把残差"溶解"掉，后者是让旧运动"滑行"出去。

它同样**不**做这些事，别误用：

- **不选动画**：它不参与上游的资产选择，只负责把"已经发生的切换"接顺。数据库挑错姿势，它只是平滑地滑进那个错误姿势，不会纠正。
- **不修正姿势来源**：Aim Offset、Warping、IK 该给的姿势还是它们给，Dead Blending 只处理这些结果之间的切换缝。
- **不是持续的 A/B 混合**：切换之后它不再采样旧动画，旧动画只以"切换瞬间快照"的形式存在。它和 Inertialization 一样，只在切换那一刻抓一次残差，之后每帧只求新姿势。

## 核心思想：外推旧动画，再交叉淡化

整个节点浓缩成两步：

1. **切换瞬间抓速度**：用前两帧输出快照，算出旧动画每个骨骼的瞬时速度（平移速度、旋转角速度、缩放速度）。
2. **外推 + 淡化**：把这些速度带着「指数衰减」往前积分，得到一具"如果旧动画继续播下去会长什么样"的外推姿势；再把它和新动画做普通的交叉淡化，直到控制权完全交给新动画。

关键在于"指数衰减"这个约束。如果只是拿速度无限外推，姿势会越飞越远、拧成麻花。所以外推给速度乘上一个衰减项，让速度每隔一段固定时间就减半——这段"减半所需的时间"就是源码里的 **half-life（半衰期）**。

外推位移的公式（源码里 `ExtrapolateTranslation` 的形状）大致是：

```text
C = ln2 / halfLife
p(t) = p0 + (v0 / C) · (1 − e^(−C·t))
```

可以看出两个性质：速度 `v(t) = v0 · e^(−C·t)` 确实在按半衰期衰减；而姿势不会无限外推，`t` 足够大时它收敛到 `p0 + v0/C` 这个有限位置。旋转、缩放、曲线走的是同一套积分逻辑，只是各自换到正确的空间（旋转用旋转向量，缩放默认用指数空间的 Eerp）。

### 半衰期是"自适应"的

衰减速度不是写死的，而是按「旧动画的速度正朝着新姿势去，还是背离新姿势」自动调的（`ComputeDecayHalfLifeFromDiffAndVelocity`）：

```text
halfLife = clamp( 平均半衰期 · (目标−源) / 速度 , 最小半衰期 , 最大半衰期 )
```

- 旧动画的速度方向**正好朝着新姿势、且相对差距很慢**：比值大 → 半衰期取到最大 → 衰减慢，让这股"本来就顺路"的动量多滑一会儿。
- 速度方向**背离新姿势、或速度太快**：比值小甚至为负 → 半衰期被压到最小 → 快速衰减，避免外推姿势飞出合理范围、把骨骼扯断。

一句话：**顺路的动量让它多跑，逆路的动量赶紧掐掉**。这就是 Dead Blending 能比"呆板地往目标拽"更自然的原因。

## 关键流程

按源码的执行顺序走一遍。

**1. 监听请求。** 它和 Inertialization 共用同一套请求机制——上游节点（Motion Matching、支持惯性化请求的 Blend 节点等）通过 `IInertializationRequester` 消息发起一次"惯性化请求"。Dead Blending 用自己的 requester 把这些请求收进队列，多个请求同时到达时取时长最短的那个。请求可以带 `Tag` 做定向过滤，只命中同名的节点。

**2. 切换帧记录状态（InitFrom）。** 拿到最短请求后，用最近两帧输出快照算出每个骨骼的速度，按 `Maximum*Velocity` 上限截断（防止单帧异常帧差把速度炸飞），再用上面那条公式结合"目标姿势"算出每根骨骼、每个分量的半衰期。曲线的值、速度、半衰期也一并记录；根运动速度则直接取快照里的根运动增量除以帧间隔。

**3. 每帧外推并淡化（ApplyTo）。** 之后每一帧：先外推出旧动画此刻的"幽灵姿势"，再算一个混合系数 `Alpha = 1 − AlphaToBlendOption(归一化时间, 混合模式, 自定义曲线)`，把目标姿势朝外推姿势 `Lerp` 过去。`t=0` 时 `Alpha=1`，输出完全等于外推姿势——而外推姿势在 `t=0` 时恰好就是记录下来的旧姿势，所以切换帧是连续的，没有 Pop；`t=时长` 时 `Alpha=0`，输出完全等于新动画。旋转额外做了"最短弧方向"跟踪，防止外推转远后突然从另一侧绕回来。

**4. 曲线和根运动同步处理。** 曲线同样被外推后与新值淡化；根运动则把旧动画的根速度外推一帧的位移，再与新动画的根运动增量淡化，最后写回属性流。这一步让胶囊位移在切换后也不会突然改速。

**5. 到时收尾。** 归一化时间超过最长时长（各骨骼时长取最大）后 `Deactivate`，清空记录。另外还有几个保险：检测到传送直接重置；`a.AnimNode.DeadBlending.Enable` 这个控制台变量可以整体关掉节点方便对比；编辑器里 `bShowExtrapolations` 能只看那具外推幽灵姿势、不看混合结果，用来调半衰期参数很直观。

## 和 Inertialization 的选择顺序

Dead Blending 不是"更好的 Inertialization"，而是同一族里的另一种配方。它们的共同点是：都只在切换瞬间抓一次残差，之后每帧只求新姿势；都靠同样的请求机制触发。真正的差别在"怎么消化残差"：

| | Inertialization | Dead Blending |
|---|---|---|
| 消化方式 | 往目标姿势上叠一个衰减差量（固定五次曲线） | 外推旧动画，与目标做交叉淡化 |
| 衰减形状 | 固定，不随请求改变 | 可用混合模式 / 自定义 Blend Curve 塑造 |
| 根运动 | 施加源与目标的**速度差**，衰减到零 | 外推旧动画的**根速度**，淡化到新动画 |
| 缩放插值 | 线性 | 默认指数 Eerp，可切回线性 |

最值得记的一点是**衰减形状可控**：Inertialization 的残差是一条固定的、保证速度和加速度连续的五次多项式；Dead Blending 因为本质是交叉淡化，可以直接套 `DefaultBlendMode` 和 `DefaultCustomBlendCurve`（请求里也能带混合模式），把"旧动画交棒给新动画"的节奏捏成你想要的样子——比如一开始快速交棒、尾巴慢慢放手的 ease-out 曲线，正是"快速起效、慢速收尾"的手感。这也解释了为什么它适合**反复启停的动态姿势层**：Aim Offset 这种频繁开合的小层，切换缝很小但很多，用一条自定义曲线把每次接缝快速抚平、再慢慢松手，比固定曲线的残差更顺手。

实际落地时的选择顺序，建议这样：

1. **先用 Inertialization 建基线。** 它是同族里更成熟、文档和社区经验更充分的一支，先把切换 Pop 消灭掉。
2. **确认问题出在"过渡"本身。** 如果调完姿势选择、Warping、IK 之后残影/抖动还在，再怀疑是过渡的手感问题，而不是姿势本身错了。
3. **需要自定义衰减形状、或想让旧运动"滑行"而不是"被拽回"时，再上 Dead Blending。** 记住它不是修复数据库误选的工具——挑错了姿势，换哪个节点都只是"平滑地错"。

## 常见问题

- **节点"没反应"**：先看 `a.AnimNode.DeadBlending.Enable` 是否为 1（默认开），再确认上游真的发出了惯性化请求、且请求的 `Tag` 与节点匹配。多个请求同时到达时只取时长最短的那个，别把时长设成 0。
- **切换后姿势"漂"、像喝醉**：多半是半衰期太长或速度上限太大，外推滑得太久。把 `Extrapolation Half Life` 的均值/最大值往下压，或收紧 `Maximum*Velocity` 上限。
- **外推姿势被硬拽回、过渡发紧**：半衰期被压到最小导致的。检查旧动画速度方向是否被误判为"背离"（比如姿势差值和速度符号的换算），必要时放宽最小半衰期。
- **缩放表现怪**：这个节点默认用指数 Eerp 处理缩放（与引擎其余部分默认的线性 Lerp 不一致），是刻意的行为。如果你有负缩放或想要引擎一贯的线性手感，勾上 `Linearly Interpolate Scales`。
- **根运动切换时跳一下**：确认上游确实写了根运动属性（同 Inertialization 的前提）。想单独看外推效果，开 `bShowExtrapolations`（仅编辑器）对比。
- **反复启停时表现与 Inertialization 不同**：两个节点的防抖策略不同——Inertialization 有"deficit"机制应对被打断的惯性化，Dead Blending 则是在每次新请求时直接从当前快照重新记录。切身体会这两者的差别，最好的办法就是同一段动画分别挂两个节点对比。

## 参考资料

- [Dead Blending 背景原理（Daniel Holden）](https://theorangeduck.com/page/dead-blending)
- [Dead Blending Node in Unreal Engine（同作者，针对 UE 节点的说明）](https://theorangeduck.com/page/dead-blending-node-unreal-engine)
- [FAnimNode_DeadBlending（官方 API 参考）](https://dev.epicgames.com/documentation/unreal-engine/API/Runtime/Engine/Animation/FAnimNode_DeadBlending)
- [FAnimNode_Inertialization（官方 API 参考）](https://dev.epicgames.com/documentation/unreal-engine/API/Runtime/Engine/FAnimNode_Inertialization)
- [Inertialization: High-Performance Animation Transitions in Gears of War（GDC 2018）](https://www.gdcvault.com/play/1025165/Inertialization)
- 引擎源码（UE 5.8）：`Engine/Source/Runtime/Engine/Classes/Animation/AnimNode_DeadBlending.h`、`Engine/Source/Runtime/Engine/Private/Animation/AnimNode_DeadBlending.cpp`；对比用 `AnimNode_Inertialization.h/.cpp`；编辑器节点 `Engine/Source/Editor/AnimGraph/Public/AnimGraphNode_DeadBlending.h`

> 说明：Dead Blending 是较新的实验性节点，本文基于 UE 5.8 源码撰写。部分字段默认值、请求合并细节与 Inertialization 的差异可能随版本调整，跨版本使用前请在目标引擎版本核对。
