# Kai Locomotion：Pivot——反向输入的预转向与衔接

> Pivot 把「仍沿旧方向滑行 + 已输入新方向」拆成 Pre-Pivot 和 Post-Pivot 两段：Pre-Pivot 用负值距离曲线匹配到预测的速度换向点，Post-Pivot 用正值距离曲线从换向点重新起步。贯穿始终的前提是——反向输入不是「停止 + 起步」，而是一个以速度换向点为界的两段事件。

## 解决的问题

反向输入时，角色同时保有旧速度和新反向加速度。若立即播放新方向起步，身体和脚步会忽略惯性；若只播放停止，又会丢失反向动作的连续性。Pivot 需要把这段「反向滑行 → 换向 → 重新加速」拆成两段来表现。

## 全貌：一个前提，两步走，两级资源

理解 Pivot，先记住一个前提：**它把反向输入拆成两段，各用一条曲线**——

- **Pre-Pivot** 复用「停止」的机制：动画距离曲线是**负值**（0 = 换向点，负值 = 离换向点还差多少），用预测的换向距离 `GetTimeAtDistance(-PivotDistance)` 定位。
- **Post-Pivot** 复用「起步」的机制：距离曲线是**正值**，每帧用 `AdvanceTimeByDistanceMatching()` 按位移推进。
- 两段共享一个总目标转角 `PivotDir`，Pre 和 Post 各自承担一部分，由旋转匹配分段写出。

在这个前提下，Pivot 分两步走，资源分两级：

1. **触发**：移动组件判定「加速度与速度反向」并设 `bIsPivoting`，角色数据用 `bCanPivot` 作资格门。
2. **Pre-Pivot → Post-Pivot**：先减速到换向点，再从换向点重新起步，最后交回 Loop。
3. **硬 / 软两级资源**：有专门 Pivot 资源走硬 Pivot（有 `PivotTime` 切点）；没有时用「旧方向 Stop + 新方向 Start」拼软 Pivot。

## 工作方式

### 触发：移动组件判定 + 资格门

移动组件在 `PhysWalking()` 里判定反向：当加速度与速度的点积 `< -0.2`（夹角约大于 101°，即反向）、速度仍大于半速、且正在加速时，置 `bIsPivoting = true`；当加速度与速度重新同向或不再加速时清除。

角色数据更新再加一道资格门：

- 靶向模式、蹲伏或 Linked Layer 切换时，直接清除 `bIsPivoting` 和 `bCanPivot`，防止用旧模式的上下文延续反向动作。
- 只有速度与加速度重新对齐（点积 `≥ 0`）且仍在移动时，才重新武装 `bCanPivot = true`。

这道门的意义是：一次反向触发 Pivot 后，必须经过「速度再次对齐」才允许下一次 Pivot，避免连续反向输入造成的状态反复。

### Pre-Pivot：用负值曲线减速到换向点

进入 Pre-Pivot 时，Kai 做三件事——定方向、选资源、定位相位：

1. **定方向**：`PivotDir = |局部加速度方向| × RotationDirection`（新输入方向，符号定左右）；旧速度方向取 `PivotInputCardinalDir` 的**相反卡迪纳尔方向**。
2. **选资源**：`SelectPivotSet()` 按优先级选——非靶向且启用旋转 Pivot → 180° 旋转 Pivot（按左右 + 左右脚）；否则有四向 Strafe Pivot → 按旧速度方向选；两者都没有 → 软 Pivot 回退（见下）。
3. **定位相位**：预测到换向点的距离 `PivotDistance = PredictGroundMovementPivotLocation().Length()`，再用 `SetExplicitTime(GetTimeAtDistance(-PivotDistance))` 把 Pre-Pivot 动画定位到「还剩 `PivotDistance` 到换向点」的那一帧。

每帧更新时，只要仍在加速，`CanDistanceMatch = bIsPivoting && bIsAccelerating` 成立，重算换向距离交给 `DistanceMatchToTarget()`；硬 Pivot 把时间 clamp 到 `PivotTime`，软 Pivot clamp 到动画结尾。

带旋转的 Pre-Pivot 同时跑旋转匹配：它承担总转角的第一段，`TargetAngle = PivotDir × PrePivot 在 PivotTime 处的旋转完成度`。

### Post-Pivot：用正值曲线从换向点重新起步

进入 Post-Pivot 时，起点由资源决定：硬 Pivot 从 `PivotTime` 继续，软 Pivot 从动画开头（0 秒）开始。之后与起步几乎一致：

- 每帧用 `AdvanceTimeByDistanceMatching()` 按本帧位移推进；
- 距离曲线值小于 `2` 时临时放宽播放速率上限；
- 带旋转的 Post-Pivot 继续跑旋转匹配，承担剩余转角：`TargetAngle = PivotDir × (1 − PrePivot 在 PivotTime 处的旋转完成度)`。

两段旋转匹配的 `TargetAngle` 加起来正好等于 `PivotDir`，把一次反向的转身拆成「Pre 段转一部分 + Post 段转剩下的」分别落到两段动画上。切出到 Loop 的条件与起步同源：非旋转且第一步完成后旋转偏移过大或速度方向已变、步态改变、或动画接近结尾。

### 硬 Pivot 与软 Pivot 回退

`PivotTime` 是硬 Pivot 的 Pre / Post 切点：Pre-Pivot 播到 `PivotTime`，Post-Pivot 从 `PivotTime` 起播。软 Pivot 没有这个切点——它用「旧方向的 Stop 动画当 Pre-Pivot、新方向的 Start 动画当 Post-Pivot」拼出连续性，Pre-Pivot 播完整个 Stop，Post-Pivot 从 Start 开头开始。

软 Pivot 不是失败，而是资源不足时的设计回退：视觉上不如专门 Pivot 动作完整，但仍保持了「减速 + 重新起步」的连续语义。

## 实现要点

- Pivot 资源选择同时需要旧速度方向和新输入方向，二者不可互换；`VelocityCardinalDirection`（旧）与 `InputCardinalDirection`（新）在软 Pivot 里分别决定 Stop 和 Start 的选择。
- 专门 Pivot 的 `PivotTime` 是 Pre / Post 切点；软 Pivot 没有该切点，Post-Pivot 从动作开头开始。
- Pre-Pivot 只有 `bIsPivoting && bIsAccelerating` 时继续距离匹配；Post-Pivot 则要求仍在加速，否则早退。
- 带旋转的 Pre / Post 动作把自定义旋转交给 `EAnimRot`；非靶向的非旋转前向 Post-Pivot 可用 `EInterpolate`。
- 换向点预测来自 `PredictGroundMovementPivotLocation()`：按速度沿加速度的分量与 GroundFriction 外推「速度换向」时刻，再做胶囊 trace 收敛到可行走表面。

## 常见问题

- **用一个「180° 转身」资源同时处理所有反向。** 靶向横移、非靶向反向、不同速度和不同落脚脚相的输入都不同，资源选择已有这些维度。
- **把软 Pivot 当成失败。** 它是资源不足时用 Stop + Start 保持运动连续性的设计；只是在视觉上不如专门 Pivot 动作完整。
- **在模式切换后沿用 Pivot。** 靶向、蹲伏或 Linked Layer 切换时，角色数据会清除 Pivot 资格；状态图也应允许重新判定，而非续播旧动画。
- **把 Pre-Pivot 当起步、Post-Pivot 当停止。** 正好相反：Pre-Pivot 用负值曲线像停止，Post-Pivot 用正值曲线像起步。

## 相关主题

- [03 地面移动的输入与朝向模式](./03_地面移动的输入与朝向模式.md)
- [17 地面状态：过渡条件与交棒](./17_地面状态_过渡条件与交棒.md)
- [04 起步：靶向与非靶向的动画选择](./04_起步_靶向与非靶向的动画选择.md)
- [06 停止：距离匹配与左右脚选择](./06_停止_距离匹配与左右脚选择.md)

## 参考资料

- Pivot 状态初始化、预测距离和 Rotation Matching：`I:/UnrealProjects/GASP/Plugins/Kai_Locomotion/Source/KaiLocomotion/Private/Animation/KLSBaseLinkedAnimInstance.cpp:356-568`
- 站立/蹲伏 Pivot 资源选择与软 Pivot 回退：`I:/UnrealProjects/GASP/Plugins/Kai_Locomotion/Source/KaiLocomotion/Private/Data/KLSDataAssets.cpp:252-381`
- Pivot 资格和状态更新：`I:/UnrealProjects/GASP/Plugins/Kai_Locomotion/Source/KaiLocomotion/Private/Library/KLSLocomotionBlueprintLibrary.cpp:159-178`
- 换向点预测：`I:/UnrealProjects/GASP/Plugins/Kai_Locomotion/Source/KaiLocomotion/Private/Library/KLSLocomotionBlueprintLibrary.cpp:678-719`
- 移动组件反向判定：`I:/UnrealProjects/GASP/Plugins/Kai_Locomotion/Source/KaiLocomotion/Private/Character/KLSCharacterMovementComponent.cpp:319-337`
