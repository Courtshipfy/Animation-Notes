# 角色运动相机系统

> 记录角色运动相机（3C 中的 Camera）如何取景、跟随和防穿墙，以及它如何与角色朝向和动画姿势耦合。

新增专题时参考仓库的 [笔记模板](../_templates/topic.md)。

## 专题索引

- [01 相机系统概览与 3C 定位](./01-相机系统概览与3C定位.md)：相机在 3C 中的职责，以及它为什么要和角色朝向解耦。
- [02 第三人称相机的核心机制](./02-第三人称相机的核心机制.md)：Pivot 锚点、旋转延迟、位置延迟和 Camera Behind Character。
- [03 碰撞检测与视角切换](./03-碰撞检测与视角切换.md)：防穿墙 Trace、第一/第三人称与 FOV 过渡。
- [04 ALS 相机实现](./04-ALS相机实现.md)：ALS v4 的 PlayerCameraManager 与 ALS Refactored 的 CameraComponent。
- [05 相机与动画的耦合](./05-相机与动画的耦合.md)：Control Rotation 如何进入 ViewState，以及 RotationMode 与相机的配合。

## 范围

本大类以动画表现为核心，相机部分只记录与角色移动、朝向和动画表现直接相关的内容：

- 视角来源（Control Rotation）如何与角色朝向解耦。
- 转动手感（旋转延迟、位置延迟）与防穿墙。
- 相机如何反过来驱动动画（ViewState、瞄准分层、取景随姿态变化）。

玩法镜头（过场相机、相机震动、轨道镜头等）不在本大类范围。

## 来源与后续

- 当前内容来自 Advanced Locomotion System（v4 / Community 与 ALS Refactored）。
- KAI Locomotion、Motion Matching 的相机方案待后续整理。
