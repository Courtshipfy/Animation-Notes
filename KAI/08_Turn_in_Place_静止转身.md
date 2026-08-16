# Kai Locomotion：Turn in Place——静止转身

> Turn in Place 只在靶向且未加速时补偿控制器与角色的朝向差；旋转曲线既推进动画，也把角色 Yaw 交给移动组件同步更新。

## 解决的问题

靶向角色站立时，镜头/控制器可能持续转动，而角色不应无限保持扭腰姿势。直接将 Actor 朝向瞬移到控制器方向会破坏脚底接触和网络表现；Turn in Place 用一段带旋转曲线的动作完成这段补偿。

## 工作方式

角色数据更新时，只有靶向模式才考虑 Turn in Place：在当前 TIP 权重为 0 的前提下，控制器与 Actor 的 Yaw 偏差绝对值达到 60°，或已有 `StrafeAlpha` 时，设定 `bTurnInPlace`；一旦存在加速度则立即清除。

进入站立 TIP 后，系统以 `AimOffsetRotation.Yaw` 选择左/右 90°动画。若上一段转身已经有 `StrafeRotationAlpha`，它按旋转曲线从相应进度续播；否则从约 1% 旋转进度开始。状态设为 `EAnimRot`，每帧从旋转曲线读取进度写回 `StrafeRotationAlpha`；02 中的移动组件据此更新 `CustomRotationYaw`。

进度接近完成，或控制器转动方向与初始转向相反时，TIP 进入 Recovery：进度清零、旋转模式改为 `EHold`，并从旋转曲线完成位置播放恢复段。

## 实现要点

- 站立选择器目前只返回左/右 `90` 资源；不要仅凭资源目录存在 180°动作就假设它一定由此路径使用，需在蓝图和数据资产中确认。
- Turn in Place 依赖旋转曲线：它决定初始定位、每帧完成度和 Recovery 起点。
- `ControlRotationSpeed` 用于检测转向方向是否反转，防止玩家迅速反向看时继续播已经失效的转身。
- 蹲伏有独立的 Turn in Place 选择和播放速率，详见 09。

## 常见问题

- **非靶向也启用 TIP。** 当前角色数据逻辑在非靶向时明确关闭 `bTurnInPlace`；角色应由移动朝向逻辑转身。
- **只播放动作、不更新 Custom Rotation。** TIP 的视觉旋转和胶囊体的实际 Yaw 必须通过 `EAnimRot` 与移动组件保持一致。
- **用固定时间触发 Recovery。** Kai 以旋转曲线完成度和转向反转作为条件，固定时长在不同动画或播放速率下会失配。

## 相关主题

- [02 角色移动组件：旋转与网络同步](./02_角色移动组件_旋转与网络同步.md)
- [03 地面移动的输入与朝向模式](./03_地面移动的输入与朝向模式.md)
- [09 蹲伏地面状态](./09_蹲伏地面状态.md)

## 参考资料

- TIP 触发条件：`I:/UnrealProjects/GASP/Plugins/Kai_Locomotion/Source/KaiLocomotion/Private/Library/KLSLocomotionBlueprintLibrary.cpp:180-203`
- 站立 TIP 与 Recovery：`I:/UnrealProjects/GASP/Plugins/Kai_Locomotion/Source/KaiLocomotion/Private/Animation/KLSBaseLinkedAnimInstance.cpp:128-197`
- 左右 TIP 资源选择：`I:/UnrealProjects/GASP/Plugins/Kai_Locomotion/Source/KaiLocomotion/Private/Data/KLSDataAssets.cpp:382-392`

