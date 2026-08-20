# UE5 现代动画节点

> 这些工具为现有动画提供运行时选择、时序与姿势修正；它们可以接入 ALS、状态机或 Motion Matching，但不从属于其中任何一个系统。每篇按「它解决什么问题 → 核心思路 → 关键流程 → 与谁配合 → 常见坑」讲解，先建立职责边界，再讨论组合关系。

## 专题导航

### 接触与 IK

- [Foot Placement](./Foot-Placement.md)：把脚部接触当成连续状态求解——落脚点、朝向、Plant/Swing 切换与骨盆补偿，不是「射线 + Two Bone IK」。
- [Leg IK](./Leg-IK.md)：执行已经确定的脚部目标，把 IK 脚解算回大腿—小腿—脚链；只回答「怎样够到」，不回答「脚该踩哪里」。
- [Full Body IK](./Full-Body-IK.md)：Control Rig 里的 Position Based IK 全身求解，让多个 Effector 共同影响骨盆、脊柱与四肢。

### 姿势 Warping

- [Orientation Warping](./Orientation-Warping.md)：把「下半身转向实际移动方向、上半身保持面向」拆成正反旋转，摊到根骨、脊柱和 IK 脚上。
- [Stride Warping](./Stride-Warping.md)：按实际速度与动画参考速度之比拉开/收窄跨步，改空间步幅不改时间。
- [Slope Warping](./Slope-Warping.md)：让整段步态适配坡面法线，必要时下拉骨盆容纳脚部高度差。

### 根运动与轨迹

- [Steering](./Steering.md)：逐步引导已选动画的根运动向目标轨迹靠拢，处理连续移动的跟随意图。
- [Motion Warping](./Motion-Warping.md)：组件 + Notify State + Root Motion Modifier 三件套，在 Warping Window 内对齐 Montage 根运动到场景目标。
- [Offset Root Bone](./Offset-Root-Bone.md)：管理 Mesh Root 相对胶囊的内部偏差，让网格短暂保留动画惯性。

### 时间

- [Distance Matching](./Distance-Matching.md)：用 Distance Curve 把「动画播到哪一帧」变成距离反查，让空间事件与游戏距离对齐。

### 查询数据与资源选择

- [Pose History](./Pose-History.md)：为 Motion Matching 保存跨帧骨骼姿势历史，提供「刚才怎样运动」的查询证据。
- [Trajectory](./Trajectory.md)：过去与预测未来的位置/朝向/速度采样序列，是 Trajectory Channel 的输入。
- [Chooser](./Chooser.md)：规则表驱动的对象选择，用上下文缩小可选资源，不混合 Pose、不算 IK。
- [Proxy Table](./Proxy-Table.md)：资源间接引用，运行时按上下文解析成最终动画/数据库/Montage。

### 动态姿势过渡

- [Blend Stack](./Blend-Stack.md)：面向动态姿势流的混合基础设施，持续吸收 Motion Matching 频繁改选的候选。
- [Inertialization](./Inertialization.md)：切换瞬间记录运动趋势、让残差逐帧衰减，来源动画可停止求值。
- [Dead Blending](./Dead-Blending.md)：与 Inertialization 同族的残差过渡，用 Blend Curve 塑造残差衰减形状。

Motion Matching 的数据库、Schema 与检索成本单独见 [MotionMatching 专题](../MotionMatching/README.md)。本目录只说明它与这些运行时模块的接口。

## 先按「修改对象」划分职责

```text
输入、移动状态、轨迹 ──→ 选择动画或姿势 ──→ 姿势 Warping ──→ 接触 / IK ──→ 最终 Pose
                                │                   │
                           时间可由 Distance      Montage Root Motion 可由
                           Matching 推进          Motion Warping 对齐目标
```

- **选择**：决定使用哪段动画或哪一个姿势，例如 Motion Matching、Chooser。
- **时间**：决定动画播放到哪一帧，例如 Distance Matching。
- **局部骨骼**：修改腿、脊柱等骨骼的姿势，例如 Orientation/Stride/Slope Warping、Leg IK。
- **Mesh Root / Root Motion**：分别处理网格相对胶囊的偏差，以及 Montage 到场景目标的根运动。

不要让两个模块反复修正同一对象。例如 ALS 的 Stride Blend 与 Stride Warping 同时大幅缩放步幅，或旧 Foot Lock 与 Foot Placement 同时锁同一只脚，都会产生过度补偿。接入时应先明确每层的唯一职责，再逐层用开关验证。

## 版本与插件

这些功能分布在 Animation Warping、Animation Locomotion Library、Motion Warping、Pose Search、Blend Stack、Control Rig、FullBodyIK（PBIK）、Chooser 等插件和模块中。节点名称、属性和实验性状态会随 UE5 小版本变化；本文解释的是功能边界，项目落地前仍应以所用引擎版本的节点属性、官方示例和源码为准。

## 相关主题

- [ALS Foot IK](../ALS/08-Foot-IK.md)
- [Motion Matching](../MotionMatching/README.md)
