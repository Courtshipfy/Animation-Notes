# Full Body IK 详解

> Full Body IK 解决的是「一条腿、一个目标」之外的全身问题：手脚头同时各有各的目标，而且这些目标互相拉扯。它的做法不是让你逐个调权重，而是把每条链抽象成带质量的刚体，用「质量 + 刚度 + 角度限制 + 偏好角度」让求解器自己决定骨盆、脊柱、四肢到底谁该让位，最后在一次迭代里整体折衷。

## 先厘清名字：FBIK 和 PBIK 是两个节点

「Full Body IK」是「全身 IK」这一类解法的通称。在 `Experimental/FullBodyIK` 插件里，Control Rig 其实能搜到两个节点：

- 旧节点 `FRigUnit_FullbodyIK`（菜单名 "Fullbody IK"），内核是 **Jacobian IK**（伪逆阻尼最小二乘，可选 Jacobian Transpose），头文件里已标注 `Deprecated`。
- 新节点 `FRigUnit_PBIK`（菜单名 "Full Body IK"），内核是 **Position Based Dynamics 式的约束解算器**。

Position/Rotation Stiffness、Preferred Angle、Rotation Limit 这一整套「谁让位」的语义，全部属于新节点 PBIK。因此后文除非特别说明，「FBIK」都指这个 Position Based 的全身求解节点；旧 Jacobian 节点的差别单独放在文末一节。

## 它解决的问题

普通 AnimGraph 的腿 IK（Leg IK / Two Bone IK）是**局部单链**的：给它一个脚部目标，它只在这条腿的大腿—小腿—脚内部重分配旋转，别处的骨骼一概不碰。这在「单脚去够一个已知落点」时又快又稳。

但很多场景不是单链问题：

- 双脚踩在不同高度，还要保持重心，骨盆得往中间让一让；
- 攀爬时手脚四个末端同时被约束，脊柱要跟着折；
- 一手扶墙、另一手够东西、头看向目标，多个 Effector 的目标互相矛盾。

这时如果每处各放一个 Leg IK，它们各自为政，骨盆、脊柱这些「共享的中间骨骼」没人统筹，结果就是互相打架、代偿过度。Full Body IK 就是来当这个「全身统筹者」的：把多个末端目标扔进同一个求解器，让共享的骨盆和脊柱在迭代中自然成为缓冲带。

它同样明确**不做**几件事：

- **不决定目标**：它只回答「给了这些末端目标，全身怎么摆」，不负责算出脚该踩哪、手该抓哪。目标位置仍由 Foot Placement、Trace、关卡交互等上游提供。
- **不选动画**：它修的是当前姿势，不会把走变跑。
- **不是 AnimGraph 节点**：它是 Control Rig 的 RigUnit，需要放进一个 Control Rig Graph，再由 AnimBP 的 `Control Rig` 节点驱动；在 AnimGraph 里直接搜是搜不到的。

## 核心思想：Position Based，多 Effector 在约束里折衷

FBIK 不解析地解每条链的角度，而是把问题拆成**一组带质量的刚体（Body）和一堆约束（Constraint）**，靠迭代互相逼近：

- 每个参与求解的关节对应一个刚体，它的「质量」由它挂着的子骨段长度求和而来——骨段越长越「重」，越不容易被拉动。
- 每个 Effector 产生一个 **Pin 约束**：把这个刚体朝目标位置/朝向拉。
- 相邻刚体之间产生 **Joint 约束**：把父子刚体在连接点重新粘合，并顺手执行角度限制。

所谓「Position Based」，就是每帧从输入姿势出发，让这些约束轮流修正位置和朝向，迭代若干轮后收敛。拉是施加在**末端**的，但通过相邻刚体的 Joint 约束，力会一层层传回根骨；多个 Effector 从不同分支同时往回拽，中间的骨盆、脊柱就成了天然的折衷点——这正是「从 Root 沿各条链传播、迭代让骨盆/脊柱/四肢共同让位」的机制来源。

## 关键流程：一次 Solve 做了什么

初始化阶段，节点先顺着层次结构把骨架变成求解器内部的数据：

1. 从每个 Effector 出发**向上走**到 Root，标出「参与求解的链」；不在任何链上的骨骼（如手指末端、头部之上）不参与。
2. 找到每个 Effector 的「链根」：向上走到分叉点、另一个 Effector 或求解 Root 为止。分叉点之上是共享的，分叉点以下各条链相对独立。
3. 为每条链上的骨段建立刚体，再在 Effector 与刚体、相邻刚体之间建立 Pin / Joint 约束。

运行时每帧的 `Solve` 大致是这几步：

1. **读入姿势**：把当前输入 Pose 抄进求解器，记录每根骨相对父骨的局部位姿和长度。
2. **算质量**：每个刚体的质量由它连着的骨段长度求和而来，`InvMass ≈ 1 / (mass × MassMultiplier)`。链越长越重，越难被拉走。
3. **根行为**：`PrePull`（默认）先整体预拉；`PinToInput` 把 Root 质量清零锁死，适合「只动四肢、根别动」的局部求解；`Free` 把 Root 当普通骨骼。
4. **定链边界**：按分叉点或显式的 `ChainDepth` 划出每条 sub-chain。
5. **混合目标**：Effector 的目标按 `PositionAlpha` / `RotationAlpha` 与输入位姿混合成最终目标；若 Root 被锁且不允许拉伸，目标会被夹回骨链可达长度内，避免硬拉。
6. **约束求解**：依次做 Root 预拉 → 偏好角度 → 链预拉 → 约束迭代（可选 SubIterations + 主迭代）。
7. **写回**：按 root-to-leaf 把刚体变换写回骨骼；不在求解分支的骨骼保持原样。

其中「约束求解」里有三个铺垫阶段值得单独讲：

- **Root PrePull（整体预拉）**：用所有 Effector 的目标位置对原始位置做「形状匹配」（累计形变梯度 + 极分解提取最优拟合旋转），得到一次整体的刚体平移+旋转，先把全身朝目标方向便宜地摆过去。这就是「骨盆/脊柱整体折衷」的粗粒度来源，可以按轴、按分量开关。
- **Preferred Angle（偏好角度）**：只有当链被**压缩**（Effector 朝链根方向缩短）时才生效，压缩越多转得越多。它只「暗示」关节朝哪个方向弯，不设上限。
- **Pull Chain Alpha（链预拉）**：把每个 Effector 的 sub-chain 当成一个整体，先朝目标整体旋转+平移一段。稀疏链更稳、密集链收敛更快，但对机器人手臂这类高度受限的链可能帮倒忙。

最后是主迭代：每轮里 Pin 约束把 Effector 朝目标拉（`OverRelaxation` 负责加速收敛），Joint 约束把父子刚体在连接点重新粘合并执行角度限制；不允许拉伸时，每轮结束再用 `RemoveStretch` 把骨段长度收回来。单关节每轮最大转角由 `MaxAngle` 限制，防止发散。最后一轮跑 `FinalPass`，从根到叶强制精确执行角度限制。

## 「谁让位」的四个旋钮

全身 IK 最容易踩的坑，就是「我不知道它为什么动肩膀/骨盆」。源码里，决定谁让位的其实是下面四样东西，理解它们比背参数清单有用得多：

1. **质量（骨段长度）**：默认大家质量都差不多，谁挂着的骨段长谁更「稳」。这是隐式的让位规则——重的地方先不动，轻的地方先去够。
2. **Position / Rotation Stiffness（位置/旋转刚度）**：这是最直接的开关。源码里位移响应会乘 `(1 − PositionStiffness)`、旋转响应会乘 `(1 − RotationStiffness)`，刚度拉到 1 就等价于锁死这个刚体。它回答的是「这根骨头愿不愿意让」。
3. **Rotation Limit（角度限制）**：每轴 Free / Limited / Locked，相对参考姿势。三轴锁死 = 固定骨，两轴锁死 = 铰链，Limited = 夹在 min/max 之间。它回答的是「这关节最多能弯多少」，把代偿赶去别处。
4. **Preferred Angle（偏好角度）**：不限制，只「暗示」。链被压缩时关节优先朝这个方向弯。源码注释写得很直白：**想让膝盖/手肘朝解剖正确方向弯，先用 Preferred Angle，而不是直接上 Limit**——因为 Limit 要多花迭代才能收敛。

还有两条配套手段，本质上是「把某些骨骼彻底请出求解器」：

- `ExcludedBones`：直接排除，不弯也不参与约束。源码注释明确建议用它来代替「把刚度调到 1」或「零范围的角度限制」。
- Effector 的 `StrengthAlpha`：这个 Effector 不主动拉，但链上骨骼仍会轻微抵抗其他 Effector 的拉扯，可以当「稳定器」用。

一句话总结：**质量决定谁重、Stiffness 决定谁肯让、Limit 决定让多少、Preferred Angle 决定往哪让**。盲目调「权重」是拿结果试错，调这四样才是在描述意图。

## 和 Leg IK 的边界

两者的选择标准不是「谁更强」，而是「有几条链、谁该参与」：

- **单脚够目标、其他部位不参与** → 用 Leg IK。它局部、便宜、快，只重分配一条腿的角度，不会牵动肩膀和脊柱。
- **多个末端同时有目标、需要骨盆/脊柱参与让位，或攀爬/全身交互** → 用 FBIK。它跨链折衷，但代价是贵，而且无关的骨骼也会被代偿进来。

最典型的是误用：单脚落地也上 FBIK，结果肩膀、脊柱、骨盆出现一堆多余代偿，既慢又不稳。反过来，全身约束里硬塞多个 Leg IK，共享的骨盆没人统筹，会互相打架。判断句就是：**目标只在一条腿内部消化，选 Leg IK；目标需要身体其他部分一起让，选 FBIK。**

## 旧 FBIK（Jacobian）差在哪

旧节点 `FRigUnit_FullbodyIK` 的求解思路完全不同，遇到老资产或老教程时能对上号：

- 它是 Jacobian IK：解「末端误差 → 关节角度修正」的线性系统，而不是位置约束迭代。
- 它的「谁参与」用 `Motion Strength` 表达：Linear / Angular Motion Strength 沿链深度从末端到根衰减（靠近 Effector 力度满，靠近根衰减到 Min），本质是另一种「让位分配」。
- 它的约束走 `FFBIKConstraintOption`：有 `Linear/Angular Stiffness`（按 twist/swing 分量）、`AngularLimit`（Free/Limit/Locked）、`PoleVector`（约束膝盖/手肘朝向）和 `OffsetRotation`。
- 它没有 PBIK 那套 Preferred Angle 与显式的质量概念，也没有 RootBehavior 三态。

所以不要拿 PBIK 的「Stiffness = 让位程度」去套旧节点——旧节点的 Stiffness 是约束选项里的分量缩放，语义不同。旧节点在头文件里已标 `Deprecated`，新项目应直接用 PBIK；具体废弃版本号以目标引擎核对为准。

## 常见问题

- **节点完全没反应**：先查初始化，别急着怀疑参数。源码里 Root 没设、设了多个 Root、或 Effector 没指到有效骨骼，都会让 `Initialize()` 直接失败。把 `Debug` 打开，或确认 Root 与 Effector 骨骼名拼写正确。
- **膝盖/手肘反向弯**：先加 `Preferred Angle`（便宜、不卡迭代），仍不行再上 `Rotation Limit`。不要一上来就锁死。
- **全身乱抖、求解发散**：把 `MaxAngle` 调小、减少 `Iterations`，或把 `OverRelaxation` 降回 1。
- **根骨被整体拽着走**：这是 `RootBehavior = PrePull`（默认）在起作用，它会整体移动 Root。想要「根锁死、只动四肢」，改成 `PinToInput`。
- **Effector 够不到目标时被硬拉**：默认 `bAllowStretch = false`，过远的目标会被夹回可达距离。只有想要卡通式拉长效果才开它。
- **肩膀/脊柱被手部 Effector 带歪**：用 `ExcludedBones` 把不想动的骨骼排除，或降低该 Effector 的 `StrengthAlpha`，而不是把刚度到处拉满。

## 参考资料

- [Control Rig Full-Body IK（官方文档）](https://dev.epicgames.com/documentation/en-us/unreal-engine/control-rig-full-body-ik-in-unreal-engine)
- [Full Body IK 节点参考](https://dev.epicgames.com/documentation/en-us/unreal-engine/node-reference/ControlRig/FullBodyIK)
- 引擎源码（UE 5.8）：`Engine/Plugins/Experimental/FullBodyIK/Source/`
  - PBIK 节点与解算器：`PBIK/Public/RigUnit_PBIK.h`、`PBIK/Public/Core/PBIKSolver.h`、`PBIK/Private/Core/PBIKSolver.cpp`
  - 刚体 / 约束（刚度、限制、偏好角度的实际实现）：`PBIK/Public/Core/PBIKBody.h`、`PBIK/Private/Core/PBIKConstraint.cpp`
  - 旧 Jacobian 节点：`FullBodyIK/Private/RigUnit_FullbodyIK.cpp`、`FullBodyIK/Public/FBIKShared.h`
