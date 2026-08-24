# 动画技术公共基础

记录不依赖某个 Locomotion 框架、可以在 ALS、KAI、Motion Matching 和其他动画系统中复用的核心概念。

## 姿势与分层

- [姿态叠加与动画分层](./01-姿态叠加与动画分层.md)：Override、Additive、骨骼遮罩、Anim Layer、Slot、Aim Offset 和 IK 如何组合成最终姿势。

## 根运动与移动驱动

- [动画资产的根运动规格](./03-动画资产的根运动规格.md)：原地动画与根运动动画的分工，ALS、Motion Matching、Kai 对「谁驱动位移、谁负责对齐」的不同选择。
- [Locomotion 动作资源规格对比](./04-Locomotion动作资源规格对比.md)：起步几步、循环几拍、几向、脚相与标定速度的差异；离散片段目录（ALS/Kai）与连续动捕数据库（Motion Matching）两条路线。

## IK 求解

- [IK 求解算法对比](./05-IK求解算法对比.md)：解析解（TwoBone）、CCD、FABRIK 三条路线的取舍，以及它们到 UE 节点（TwoBoneIK / CCDIK / FABRIK / Leg IK / Full Body IK）的映射。

具体系统如何落地这些概念，应继续阅读对应系统的实现专题。

- [ALS：瞄准与 Overlay](../ALS/07-瞄准与Overlay.md)
- [KAI：姿态叠加与分层实现](../KAI/11_姿态叠加与分层实现.md)
- [Motion Matching：姿态叠加与分层实现](../MotionMatching/15-姿态叠加与分层实现.md)
