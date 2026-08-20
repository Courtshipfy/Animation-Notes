# Offset Root Bone 详解

> Offset Root Bone 做的事一句话就能说清：让「网格根骨」不再每一帧死贴「胶囊」，而是保留一段动画自己的位移与转向惯性，晚一步再追上。它管理的不是角色往哪走，而是网格和胶囊之间那点「内部偏差」。

## 它解决的问题

胶囊（SkeletalMeshComponent 的变换）是角色移动的真正载体：每一帧它被 Character Movement 的输入、速度、碰撞即时驱动。但动画里那段根运动（起步、停止、转向时身体重心的惯性摆动）有自己独立的节奏。如果网格根骨永远等于胶囊，胶囊一被推、一被撞、一转向，网格立刻跟着抖——起步和停止时身体像被「钉」在胶囊上，脚刚落地重心就被拽走，表现生硬。

Offset Root Bone 就是把这层「刚性绑定」松开：**胶囊即时服从移动、碰撞、输入，网格短暂保留动画惯性**，两者之间的差距被记录成一笔「偏移」，再在合适的时机回收。

它明确**不做**几件事，理解边界才不会误用：

- **不读外部目标**：没有 Warp Target，不关心你要去哪里；它只看组件变换和上游根运动。
- **不移动 Actor**：胶囊/角色该往哪走还是移动系统说了算，它只改姿势里第 0 号根骨的变换。
- **不补碰撞检查**：默认不检查偏移会不会把网格推进障碍（可选碰撞测试只是沿偏移方向的一次形状投射，见后文）。

## 核心思想：两个变换之差

节点内部维护两套世界空间变换：

- **组件变换 ComponentTransform**：每帧从 AnimInstanceProxy 读取，就是胶囊/网格组件此刻的位置朝向。
- **模拟变换 SimulatedTransform**：节点「想让网格根骨待在哪」——它不急着等于组件，而是按模式累积、锁定或释放。

两者之差就是偏移：

```text
偏移 = ComponentTransform − SimulatedTransform
```

这行式子（源码注释里原样写着）是整个节点的心智模型：**Simulated 是网格根骨实际被放到的位置，Component 是胶囊的位置，两者差多少，网格就落后/超前多少**。求值时节点把根骨写到 `SimulatedTransform * ComponentTransform.Inverse()`（即相对组件空间的那个量），等价于把网格根骨放到 SimulatedTransform 去。

所以剩下的问题只有一个：**每一帧让 Simulated 怎么变**。答案由 `TranslationMode` / `RotationMode`（平移、旋转可分别设）决定，而每种模式其实是三件开关的组合：

1. **要不要抵消组件位移**——把组件这一帧的增量同步加进 Simulated（偏移冻结），还是不加（组件一动，偏移就吸收这笔位移）。
2. **要不要消费动画根运动**——把上游根运动计入 Simulated。
3. **要不要插值释放**——每帧把偏移往回拉一点。

## 关键流程

一帧求值的大致顺序（对平移、旋转各跑一遍）：

1. 读当前组件变换，处理传送（见常见问题）。
2. 按模式定「抵消 / 消费 / 释放」三件开关，从根运动属性流提取增量（Graph 模式），或直接用 Delta 引脚（Manual 模式）。
3. 需要抵消时，把组件这一帧的增量同步进 Simulated；需要消费时，把动画根运动乘进 Simulated。
4. 若开 On Ground，把 Simulated 平移到组件所在地面平面（由 GroundNormal 定义）上。
5. Interpolate / Release 模式做阻尼回收：每帧按半衰期式阻尼收掉偏移的一部分，值越小收得越快，且对帧率稳定。
6. 用 Max Translation / Rotation Error 把偏移钳在允许范围内。
7. 把 `SimulatedTransform * ComponentTransform.Inverse()` 累加进根骨姿势，缩放保持输入不变。
8. Graph 模式下把「没被消费」的剩余根运动写回属性流，交给移动系统继续用。

一句话：**先算出这一帧 Simulated 该待哪，再把它相对胶囊的差量写到根骨上**。

## 六个模式

`EOffsetRootBoneMode` 的六种模式，本质就是上面三件开关的组合：

| 模式 | 抵消组件位移 | 消费动画根运动 | 插值释放 | 语义 |
|---|---|---|---|---|
| Accumulate | 否 | 是 | 否 | 组件一动偏移就累积，根「停在原地」 |
| Interpolate | 否 | 是 | 是 | 累积，但每帧渐进回收 |
| LockOffsetAndConsumeAnimation | 是 | 是 | 否 | 冻结当前偏移，仍按动画走 |
| LockOffsetIncreaseAndConsumeAnimation | 是 | 是（只许缩小偏移） | 否 | 冻结，且根运动不得再放大偏移 |
| LockOffsetAndIgnoreAnimation | 是 | 否 | 否 | 完全冻结，网格跟死胶囊 |
| Release | 是 | 否 | 是 | 只回收，不再累积 |

逐条展开：

- **Accumulate（累积）**：不抵消组件位移，也不释放——胶囊怎么动，网格都不跟着挪，只吃动画根运动，于是偏移不断累积，网格根骨相对胶囊「留在原地」。通常用于起步/转向开头那几帧，先把身体惯性留足；但它**不会自己回来**，必须后面切到 Release / Interpolate 回收。
- **Interpolate（插值释放）**：和 Accumulate 一样累积，但每帧用半衰期阻尼把偏移往回拉。表现是「网格落后胶囊一段，又不断追赶」。这是节点默认模式，适合「允许短暂分离、但总要追平」的常态表现。
- **LockOffsetAndConsumeAnimation（锁定偏移 + 消费动画）**：开始抵消组件位移——Simulated 同步跟随胶囊，当前偏移被冻住；同时仍消费动画根运动。适合「偏移已经到位，接下来让动画在这层相对胶囊的偏差上继续推进」。
- **LockOffsetIncreaseAndConsumeAnimation（锁定 + 只许缩小）**：在上一档基础上加约束——只消费能让偏移变小的那部分根运动（源码把平移投影到偏移轴向、旋转把角度钳进当前偏移角范围）。结果是偏移只会缩小、不会长大。
- **LockOffsetAndIgnoreAnimation（锁定 + 忽略动画）**：既抵消组件位移又忽略动画根运动，Simulated 跟胶囊完全同步，偏移被原样冻住。想「硬锁」住当前偏差、也不让动画再改动它时用。
- **Release（释放）**：抵消组件位移（偏移不再因胶囊移动而改变），同时插值回收、忽略动画根运动。就是「把存下的偏移一笔一笔还回去」，直到网格和胶囊重合。

平移和旋转是两个独立模式，可以不同——常见是平移用 Interpolate 让它追平、旋转用 Accumulate 让转身惯性多留一会儿。

## 和谁配合

- **上游要有根运动属性流**。默认 `EvaluationMode = Graph`，它从根运动 Provider 提取当前子图累积的根运动，所以前一级通常是 Motion Matching 或启用 Root Motion 的 Sequence Player。取不到时切 Manual 模式，用 `TranslationDelta` / `RotationDelta` 引脚手喂增量（注意这两个是「每帧增量」，程序化输入要乘 DeltaTime）。
- **给 Orientation Warping 当参考空间**。节点通过图消息 `FRootOffsetProvider` 把当前 Simulated 变换广播给下游。Orientation Warping 据此把「移动方向」相对根骨当前姿势来算，而不是相对组件——这样网格已经偏开时方向修正不会打架；没有这个节点时它退回组件空间。
- **和 Motion Warping 分清边界**。两者都带 Root：Motion Warping 改「Montage 到场景目标」的外部根运动，Offset Root Bone 管「网格相对胶囊」的内部偏差。叠加时同时观察胶囊、Mesh Root、Root Motion 三层，别在同一层上修两次。
- **放在姿势修正之前**。它只动根骨（第 0 号骨），是一个「根级」节点，通常放在 AnimGraph 较前的位置，让后面所有骨骼修正都发生在已经「偏开」的根上。

## 常见问题

- **网格越拉越远、甚至穿模**：十有八九是 Accumulate（或 Lock 档）没有回收时机。Accumulate 不会自己回来，必须在停止/Idle 前切到 Interpolate 或 Release。
- **默认模式就够吗**：Interpolate 是「总在追赶」的默认，适合常规分离；要「存住不动等事件触发」用 Lock 档，要「立刻还清」用 Release。平移、旋转分别设，别一锅端。
- **网格推进墙里**：默认不检查碰撞，偏移能直接把网格塞进障碍。可选 `CollisionTestingMode`（ShrinkMaxTranslation / PlanarCollision）只是沿偏移方向做一次球形投射，只限制平移、且依赖 MaxTranslationError > 0，不是完整碰撞修正。要稳，还是控制好偏移上限和回收时机。
- **Graph 模式没反应 / 报 ensure**：默认是 Graph 模式，但上游必须真的写了根运动属性；否则切 Manual 并手喂 Delta。先确认 `EvaluationMode` 到底在哪个档。
- **传送后网格停在远处**：默认传送时把偏移「平移保留」过去；想传送后立刻对齐就开 `bResetOnTeleport`。
- **转身惯性想多留、却被一下拽平**：把旋转模式换成 Accumulate 或调大旋转半衰期；反之要快速追平就调小半衰期（越小收得越快）。
- **配合 Orientation Warping 时方向偏**：确认 Offset Root Bone 在 Orientation Warping 之前、且参考空间确实收到了它广播的根变换；顺序接反或消息没接上，方向修正会退回组件空间。

## 参考资料

- [Pose Warping](https://dev.epicgames.com/documentation/en-us/unreal-engine/pose-warping-in-unreal-engine)
- [Game Animation Sample Project](https://dev.epicgames.com/documentation/en-us/unreal-engine/game-animation-sample-project-in-unreal-engine)
- 引擎源码（UE 5.8）：`Engine/Plugins/Animation/AnimationWarping/Source/Runtime/`（`FAnimNode_OffsetRootBone` 位于 `BoneControllers/AnimNode_OffsetRootBone.h/.cpp`，模式枚举 `EOffsetRootBoneMode` 位于 `Public/AnimationWarpingTypes.h`）
