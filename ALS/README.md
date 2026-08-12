# Advanced Locomotion System

记录 ALS 中与角色移动动画直接相关的架构、数据流、状态机、姿势混合、IK 和调试方法。

新增专题时可参考仓库的 [笔记模板](../_templates/topic.md)。

## 架构

- [整体架构](./01-整体架构.md)：ALS 动画系统的模块关系和总体流程。
- [动画数据流](./02-动画数据流.md)：运行时状态如何进入动画图并形成最终姿势。
- [动画实例](./03-动画实例.md)：AnimInstance 的职责、数据分组和更新方式。

## 动画系统

- [Grounded](./04-Grounded.md)：地面移动、Idle、Cycle、Start、Stop 和 Pivot。
- [In Air](./05-In-Air.md)：跳跃、下落和落地相关的动画上下文。
- [起步、停止与转身](./06-起步停止与转身.md)：移动阶段和原地转身的选择逻辑。
- [瞄准与 Overlay](./07-瞄准与Overlay.md)：Aim Offset、上半身分层和覆盖姿势。
- [Foot IK](./08-Foot-IK.md)：脚部 IK、脚锁定和骨盆偏移。

## 实践

- [动画资产组织](./09-动画资产组织.md)：Sequence、Blend Space、Curve、Layer 和 Slot 的职责。
- [调试问题](./10-调试问题.md)：按“数据 → 状态 → 姿势 → 修正”的顺序定位问题。
