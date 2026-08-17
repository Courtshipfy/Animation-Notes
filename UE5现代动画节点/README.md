# UE5 现代动画节点

> 这些工具为现有动画提供运行时选择、时序与姿势修正；它们可以接入 ALS、状态机或 Motion Matching，但不从属于其中任何一个系统。

## 专题导航

- [01 接触与 IK](./01-接触与IK.md)：Foot Placement、Leg IK 与 Full Body IK 分别怎样确定脚的接触、解算腿链和传递全身约束。
- [02 Pose Warping](./02-Pose-Warping.md)：Orientation、Stride、Slope Warping 怎样在不增加同等数量动画资产的情况下适配方向、速度和坡度。
- [03 根运动与根骨修正](./03-根运动与根骨修正.md)：Motion Warping 与 Offset Root Bone 分别修改外部目标轨迹和 Mesh Root 相对胶囊的偏差。
- [04 Distance Matching](./04-Distance-Matching.md)：用距离曲线驱动动画时间，适配停止、Pivot、起步和落地。
- [05 查询数据与资源选择](./05-查询数据与资源选择.md)：Pose History、Trajectory、Chooser 和 Proxy Table 为姿势搜索或资产选择提供上下文。
- [06 动态姿势过渡](./06-动态姿势过渡.md)：Blend Stack、Inertialization 与 Dead Blending 如何处理频繁重定向的姿势流。

Motion Matching 的数据库、Schema 与检索成本单独见 [MotionMatching 专题](../MotionMatching/README.md)。本目录只说明它与这些运行时模块的接口。

## 先按“修改对象”划分职责

```text
输入、移动状态、轨迹 ──→ 选择动画或姿势 ──→ 姿势 Warping ──→ 接触 / IK ──→ 最终 Pose
                                │                   │
                           时间可由 Distance      Montage Root Motion 可由
                           Matching 推进          Motion Warping 对齐目标
```

- **选择**：决定使用哪段动画或哪一个姿势，例如 Motion Matching、Chooser。
- **时间**：决定动画播放到哪一帧，例如 Distance Matching。
- **局部骨骼**：修改腿、脊柱等骨骼的姿势，例如 Pose Warping、Leg IK。
- **Mesh Root / Root Motion**：分别处理网格相对胶囊的偏差，以及 Montage 到场景目标的根运动。

不要让两个模块反复修正同一对象。例如 ALS 的 Stride Blend 与 Stride Warping 同时大幅缩放步幅，或旧 Foot Lock 与 Foot Placement 同时锁同一只脚，都会产生过度补偿。接入时应先明确每层的唯一职责，再逐层用开关验证。

## 版本与插件

这些功能分布在 Animation Warping、Animation Locomotion Library、Motion Warping、Pose Search、Control Rig、FullBodyIK 等插件和模块中。节点名称、属性和实验性状态会随 UE5 小版本变化；本文解释的是功能边界，项目落地前仍应以所用引擎版本的节点属性、官方示例和源码为准。

## 相关主题

- [ALS Foot IK](../ALS/08-Foot-IK.md)
- [Motion Matching](../MotionMatching/README.md)
