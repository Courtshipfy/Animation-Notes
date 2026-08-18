# ALS 相机实现

> ALS 的两版相机共用同一套机制，区别在组织方式：v4 用全局 CameraManager，Refactored 用角色组件 + 数据资产。

## ALS v4 / Community：PlayerCameraManager

载体是自定义 `APlayerCameraManager`，蓝图里重写 `CustomCameraBehavior`（C++ 版同名函数）。每帧大体执行：

1. 取 `PivotTarget`（受控 Pawn），计算带 `PivotLagSpeed` 平滑的锚点。
2. 目标旋转 = Control Rotation；若开启 `CameraBehindCharacter` 且角色在移动，先把目标 Yaw 回正到角色后方。
3. 用 `CameraRotationLagSpeed` 平滑旋转（`CalculateAxisIndependentLag`）。
4. 期望位置 = 锚点 + 由旋转推导的偏移，再用 `CameraLagSpeed` 平滑。
5. `ThirdPersonTrace` 做防穿墙，命中则拉近。
6. 第一人称时切到 `FirstPersonCameraSocketName`，FOV 在 `ThirdPersonFOV` / `FirstPersonFOV` 间过渡。
7. 按 `DebugView` / `TraceDebug` 输出调试绘制。

关键参数（示例默认值，以具体版本源码为准）：`PivotLagSpeed ≈ 7`、`CameraLagSpeed ≈ 7`、`CameraRotationLagSpeed ≈ 12`、`ThirdPersonFOV = 90`、`FirstPersonFOV = 90`、`CameraTraceChannel = Visibility`、`FirstPersonCameraSocketName = head`。

## ALS Refactored：CameraComponent + CameraSettings

载体是挂在角色上的 `UAlsCameraComponent`，配合 `UAlsCameraSettings` 数据资产。`TickComponent` 把计算拆成阶段：锚点 → 旋转 → 距离 → 位置 → FOV，最终由 `GetViewInfo` 把 POV 交给视图目标。

相比 v4 的差别：

- 相机随角色组件走，天然跟随，且能正确处理移动平台（BasedMovement）。
- 旋转偏移用 `CameraBehavior`（Pitch / Yaw / Roll 三轴偏移），取景用 `CameraHeight`、`CameraDistance`、`CameraFov`，这些可被外部逻辑或动画驱动，实现瞄准拉近、蹲伏降相机等。
- 延迟、碰撞半径、追踪通道等参数集中到 `UAlsCameraSettings`，可跨角色复用和调参。

## 怎么选

- 学机制、看原始蓝图的第三人称实现，读 v4 的 `CustomCameraBehavior` 更直观。
- 要数据资产集中配置、移动平台支持、把相机参数交给动画 / 玩法驱动，Refactored 的组织方式更现代。

## 相关主题

- [01 相机系统概览与 3C 定位](./01-相机系统概览与3C定位.md)
- [02 第三人称相机的核心机制](./02-第三人称相机的核心机制.md)

## 参考资料

- [ALSV4_CPP `ALSPlayerCameraManager.cpp` 源码](https://github.com/Peralysis/ALSV4_CPP/blob/main/Source/ALSV4_CPP/Private/Character/ALSPlayerCameraManager.cpp)
- [ALS-Refactored Camera Component (DeepWiki)](https://deepwiki.com/Sixze/ALS-Refactored/4.1-camera-component)
- [ALS-Refactored Camera Settings (DeepWiki)](https://deepwiki.com/Sixze/ALS-Refactored/4.2-camera-settings)
- [【Advanced Locomotion System】Camera System 拆解（知乎）](https://zhuanlan.zhihu.com/p/451943770)
