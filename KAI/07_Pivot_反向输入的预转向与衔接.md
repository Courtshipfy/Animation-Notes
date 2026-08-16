# Kai Locomotion：Pivot——反向输入的预转向与衔接

> Pivot 将“仍沿旧方向滑行”与“已经输入新方向”拆为 Pre-Pivot 和 Post-Pivot，分别匹配减速到转折点、以及从转折点重新起步的表现。

## 解决的问题

反向输入不是普通的停止后起步：角色仍有旧速度，也已经有新加速度。若立即播放新方向起步，会让身体和脚步忽略惯性；若只播放停止，又会丢失反向动作的连续性。

## 工作方式

移动组件先在 `PhysWalking()` 判断加速度与速度近似反向；角色数据更新又以 `bCanPivot` 防止不合法状态持续 Pivot。进入 Pre-Pivot 时，Kai：

1. 用局部加速度方向确定 Pivot 目标角和输入卡迪纳尔方向，再取其相反方向作为旧速度方向。
2. 依据站立/蹲伏、步态、靶向、左右脚和资源可用性选择 `FKLSPivotAnimSet`。
3. 预测角色到“速度换向点”的距离，以此在 Pre-Pivot 的距离曲线上定位并持续推进；带旋转的动作同时执行 Rotation Matching。
4. 到达 Pivot 时间后进入 Post-Pivot：硬 Pivot 从该时间点开始，软 Pivot 从 0 秒开始；Post-Pivot 再以新输入方向选择动作并按位移推进，最后转入 Loop。

数据资产优先选择专门的旋转 Pivot（非靶向且已启用）或四向 Strafe Pivot。两者都没有时，才把旧方向的 Stop 动作作为 Pre-Pivot、新方向的 Start 动作作为 Post-Pivot，形成软 Pivot 回退方案。

## 实现要点

- Pivot 资源选择同时需要旧速度方向和新输入方向，二者不可互换。
- 专门 Pivot 的 `PivotTime` 是 Pre/Post 的切点；软 Pivot 没有该切点，Post-Pivot 从动作开头开始。
- Pre-Pivot 只有在 `bIsPivoting && bIsAccelerating` 时继续距离匹配；Post-Pivot 则要求仍在加速。
- 带旋转的 Pre/Post 动作将自定义旋转交给 `EAnimRot`；非靶向的非旋转前向 Post-Pivot 可使用 `EInterpolate`。

## 常见问题

- **用一个“180°转身”资源同时处理所有反向。** 靶向横移、非靶向反向、不同速度和不同落脚脚相的输入都不同，资源选择已有这些维度。
- **把软 Pivot 当成失败。** 它是资源不足时用 Stop + Start 保持运动连续性的设计；只是在视觉上不如专门 Pivot 动作完整。
- **在模式切换后沿用 Pivot。** 靶向、蹲伏或 Linked Layer 切换时，角色数据会清除 Pivot 资格；状态图也应允许重新判定，而非续播旧动画。

## 相关主题

- [03 地面移动的输入与朝向模式](./03_地面移动的输入与朝向模式.md)
- [04 起步：靶向与非靶向的动画选择](./04_起步_靶向与非靶向的动画选择.md)
- [06 停止：距离匹配与左右脚选择](./06_停止_距离匹配与左右脚选择.md)

## 参考资料

- Pivot 状态初始化、预测距离和 Rotation Matching：`I:/UnrealProjects/GASP/Plugins/Kai_Locomotion/Source/KaiLocomotion/Private/Animation/KLSBaseLinkedAnimInstance.cpp:356-568`
- 站立/蹲伏 Pivot 资源选择与软 Pivot 回退：`I:/UnrealProjects/GASP/Plugins/Kai_Locomotion/Source/KaiLocomotion/Private/Data/KLSDataAssets.cpp:252-381`
- Pivot 资格和状态更新：`I:/UnrealProjects/GASP/Plugins/Kai_Locomotion/Source/KaiLocomotion/Private/Library/KLSLocomotionBlueprintLibrary.cpp:159-178`

