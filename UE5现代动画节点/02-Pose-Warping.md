# Pose Warping：Orientation、Stride、Slope、Steering

> Pose Warping 不选择动画，也不移动 Actor；它重排当前 Pose 的局部骨骼关系，使同一段动作更贴近实际方向、速度与坡面。

## Orientation Warping：修正动画前向与移动方向的残差

Orientation Warping 是方向姿势修正节点。动画向前跑、角色却相对身体斜向移动时，它让下半身步态朝实际移动方向偏转，同时尽量维持上身面向。

它的原理是输入局部移动方向的有符号角度，旋转左右 IK Foot 的运动关系，并把部分反向扭转分配给 Spine。IK Foot Root 是下半身参考空间；Spine 列表决定躯干承担多少补偿；随后 Leg IK 才将修改后的 IK Foot 目标解算回实际腿链。它修的是当前 Pose 的方向残差，不会驱动 Actor 转向。

典型应用是用一段向前循环覆盖中等角度的斜向跑、瞄准移动，或修正 Motion Matching 选中的姿势。ALS 的方向 Blend Space 已经靠资产承担方向时，应明确谁是主方案；同一动作被两者重复偏转会导致髋部和脊柱过扭。大角度后退、横移仍应使用对应资产。

## Stride Warping：修正一步跨多远

Stride Warping 是步幅姿势修正节点。它调整双脚的空间间距，使动画标定速度与角色实际移动速度不同的时候，脚仍能覆盖合理地面距离。

它的原理是根据实际 Locomotion Speed 与动画的参考 / Root Motion Speed 得到步幅比例，近似为 `Stride Scale ≈ Locomotion Speed / Root Motion Speed`。大于 1 拉开跨步，小于 1 收窄跨步；它改变空间长度，不改变动画时间。Play Rate 改的是节奏，因此两者可有限配合、却不应同时无边界补偿同一个速度误差。

典型应用是一套走跑循环覆盖有限速度区间，减少单靠提高播放速率造成的碎步。ALS 的 `StrideBlendAmount` 与它属于同一层解决方案：基线速度下比例应接近 1；若此时异常，应先检查动画参考速度、根骨数据或速度单位。极端速度仍须切换合适动画。

## Slope Warping：让整段步态适应坡面

Slope Warping 是坡面姿势修正节点。它让下半身移动姿势随地面法线和移动方向变化，而不是只把最终脚掌旋转到地面上。

它的原理是重排骨盆、腿和 IK Foot 的相对位置，使上坡时保留抬腿量、下坡时维持重心和腿部压缩；需要时下拉骨盆以容纳脚部高度差。它处理一整个运动周期怎样适应坡度，单脚接触系统只处理这只脚这一帧该踩哪里。

典型链路是平地循环在缓坡、楼梯和不规则地面上的第一层适配：`Slope Warping → Foot Placement`，先改整体步态，再确定每脚落点与支撑。先锁脚再改下半身会重新破坏落脚结果；它也不能取代地面检测。

## Steering：逐步引导根运动轨迹

Steering 是现代 Motion Matching 链路中的根运动引导能力，让已选动画在播放过程中逐步向目标轨迹靠拢。

它的原理是比较当前动画根运动趋势与目标轨迹，在剩余播放时间内逐步施加方向引导；它主要处理动画运动轨迹，不同于 Orientation Warping 对腿与脊柱局部 Pose 的修正。原始根运动与目标差异过大时，仍需要重选动画或切换状态。

典型应用是 Game Animation Sample 在 Motion Matching 的 Blend Stack Graph 中用它让连续移动跟随输入意图。角色必须在交互点精确结束时，应使用 [Motion Warping](./03-根运动与根骨修正.md)，而不是 Steering。

## 参考资料

- [Pose Warping](https://dev.epicgames.com/documentation/en-us/unreal-engine/pose-warping-in-unreal-engine)
- [Game Animation Sample Project](https://dev.epicgames.com/documentation/en-us/unreal-engine/game-animation-sample-project-in-unreal-engine)
