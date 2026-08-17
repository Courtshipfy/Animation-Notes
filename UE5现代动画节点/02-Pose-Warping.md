# Pose Warping：方向、步幅与坡度

> Pose Warping 保留当前动画的基本节奏，再在运行时改变局部骨骼的空间关系，让它更接近实际移动条件。

## 三类 Warping 分别修正什么

| 节点 | 输入的差异 | 主要修改 | 适用场景 |
| --- | --- | --- | --- |
| Orientation Warping | 实际移动方向与动画前向的夹角 | 腿部 IK 骨骼的朝向，并把部分扭转分配到脊柱 | 用较少方向动画覆盖斜向移动 |
| Stride Warping | 实际移动速度与动画标定速度的比例 | 双脚的跨步距离 | 同一套走跑动作适配一段速度范围 |
| Slope Warping | 当前坡面与角色移动方向 | 下半身相对坡面的整体姿势 | 上下坡时保留合理的步态与腿部弯曲 |

它们都在 Pose 的局部骨骼空间中工作，不会替角色移动组件转动 Actor，也不能替代正确的 Root Motion 或碰撞移动。

## Orientation Warping：让移动方向不只由动画资产决定

动画可以是向前跑，而角色速度相对身体前向偏右。Orientation Warping 将这个夹角用于重新定向腿部运动，并把一部分扭转交给脊柱，从而避免下半身完全朝前、速度却明显向侧方的割裂感。

它减少的是中间方向动画的需求，不是无限扩大单一动画的覆盖范围。角度过大时，髋部、膝盖和上半身扭转仍会失真；后退、横移或持枪等具有强语义的动作，仍应保留对应资产或状态。

对于 ALS，`MovementDirection`、Rotation Yaw Offset 和方向 Blend Space 是资产选择与曲线驱动的方案；Orientation Warping 是选择之后的姿势修正。两者可共存，但必须决定谁是主方案，避免方向动画已充分偏转后又被节点重复扭转。

## Stride Warping：修正空间步幅，不等于改播放速度

Stride Warping 用角色移动速度与动画根运动的标定速度估算步幅比例，可概括为：

```text
Stride Scale ≈ Locomotion Speed / Root Motion Speed
```

角色更快时扩大双脚间的距离，更慢时缩小距离。它主要解决“空间上跨得太大或太小”；播放速率主要解决“时间节奏太快或太慢”。两者可以在有限范围内配合，但不应同时无边界地补偿速度。

使用前要确认动画的 Root Motion Speed 或参考速度有效。原地（in-place）动画、错误的根骨速度、以及没有重新标定的 ALS `StrideBlendAmount`，都会使比例失真，表现为碎步、滑步或腿部过度伸展。

## Slope Warping：先适配整段步态，再处理落脚点

Slope Warping 使用坡度与移动方向调整下半身的整体运动姿势。它解决的是“平地循环跑步直接放到斜坡上”时的髋部高度、抬腿量与重心方向不自然，而不只是把脚掌旋转到地面法线。

在不规则地面上，一个常见分工是：Slope Warping 先塑造整体上下坡姿势，Foot Placement 再根据每只脚的实际接触点做末端放置与骨盆补偿。只用脚部 IK 虽然能碰到地面，却常留下身体仍在平地跑、膝盖压缩不合理的问题。

## 与 Steering 的边界

Steering 在现代 Motion Matching 链路中用于在播放期间逐步引导根运动方向，使已选动画更接近目标轨迹。它改变的是动画运动轨迹的引导，不能简单视为 Orientation Warping 的同义词；后者主要修正当前姿势的方向关系。不同 UE5 版本的接口与示例组织变化较快，使用时以目标版本的 Game Animation Sample 为准。

## 调试顺序

1. 只保留基础动画，确认速度、根骨朝向和 IK Foot Bone 没有问题。
2. 单独启用 Orientation、Stride、Slope 中的一种，观察其输入值与关节极限。
3. 再加入 Foot Placement 或 Leg IK，确认最终接触没有被 Warping 二次破坏。

## 相关主题

- [接触与 IK](./01-接触与IK.md)
- [ALS 步态混合与播放速率](../ALS/04.7-步态混合与播放速率.md)
- [ALS 移动方向与 Rotation Yaw Offset](../ALS/04.3-移动方向与RotationYawOffset.md)

## 参考资料

- [Pose Warping](https://dev.epicgames.com/documentation/en-us/unreal-engine/pose-warping-in-unreal-engine)
- [Game Animation Sample Project](https://dev.epicgames.com/documentation/en-us/unreal-engine/game-animation-sample-project-in-unreal-engine)
