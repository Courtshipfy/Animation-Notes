# ALS 瞄准与 Overlay

> 瞄准和 Overlay 将上半身姿势叠加到基础移动姿势，同时保持下半身的移动表现。

## 工作方式

1. 计算控制器旋转与角色旋转的差值，得到 Aim Yaw 和 Aim Pitch。
2. 将瞄准角输入 Aim Offset，生成上半身倾斜和转向姿势。
3. 以 Grounded 或 In Air 产生的全身姿势作为基础。
4. 使用 Layered Blend Per Bone、Additive Pose、Anim Layer 或 Anim Slot 叠加武器、交互和受击动作。
5. 根据 Overlay State 和 Layer Weight 控制脊柱、头部、手臂及手部 IK 的影响范围。

## 设计重点

移动循环和瞄准动作的变化频率不同。将下半身移动与上半身瞄准分层，可以复用同一套移动动画，也能限制局部动作对脚步的影响。

## 常见问题

- 骨骼混合范围过大，导致脚步或骨盆被上半身动作影响。
- Additive Pose 的参考姿势不匹配，造成姿势偏移。
- Overlay 权重没有在 Grounded 和 In Air 之间正确切换。
- Anim Slot 的组名或播放位置不一致，导致蒙太奇没有叠加到预期层。

## 相关主题

- [Grounded](./04-Grounded.md)
- [Foot IK](./08-Foot-IK.md)
- [动画资产组织](./09-动画资产组织.md)
