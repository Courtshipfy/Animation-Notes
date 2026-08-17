# Distance Matching：按距离而不是时间推进动画

> 它将“动画现在应播放到哪一帧”变成反查问题：给定距离，在曲线上找对应姿势。

## Distance Matching：距离驱动的时间重参数化

Distance Matching 是 Animation Locomotion Library 的一套工作流，不是普通输入 Pose → 输出 Pose 节点。它依赖 Animation Sequence 的 Distance Curve、Sequence Evaluator 和 Anim Node Function，让动画由距离变量而非 Delta Time 推进。

它的原理是 Distance Curve 记录 `动画时间 t → 约定距离 d`。运行时由移动模型得到目标距离 `D`，在曲线上反查 `d(t) ≈ D` 的时间 `t`，并写入 Sequence Evaluator 的 Explicit Time。动画仍播放原有关键帧，只是播放进度被距离重参数化。曲线需以可运行时索引的压缩方式保存，官方流程使用 Uniform Indexable Animation Compression。

典型应用是停止时以剩余制动距离选择停步相位，Pivot 时以到转向点的距离控制蹬地，落地时以到地面的高度挑选缓冲姿势，起步时以已前进距离同步动作。链路是 `距离预测 / Trace → 节点函数反查 Distance Curve → Sequence Evaluator`。它不预测距离，也不选择动画；这些仍属于移动模型和状态选择。

## Distance Curve：把空间事件固定到正确姿势

Distance Curve 是动画“每个姿势对应多少位移”的查找表；它与只调整时间倍率的 Play Rate 不同。

它的原理是 Play Rate 会让所有帧一起加速或减速，不能保证关键脚相位恰在剩余距离为零时发生；Distance Matching 直接选择对应距离的帧，让空间事件同游戏距离一致。曲线方向、零点和单位必须与输入变量一致：曲线若从最大剩余距离下降到 `0`，运行时就必须输入同定义的剩余距离。

典型用法是先用固定距离检查反查时间和输出姿势，再接入制动预测或落地 Trace。调试时记录 `D`、反查时间 `t`、曲线值 `d(t)`：三者不一致通常是曲线定义或单位错误；三者正确而动画不动，通常是 Explicit Time 没有写入 Evaluator。

## 参考资料

- [Distance Matching](https://dev.epicgames.com/documentation/en-us/unreal-engine/distance-matching-in-unreal-engine)
- [Animation Locomotion Library](https://dev.epicgames.com/documentation/en-us/unreal-engine/API/Plugins/AnimationLocomotionLibrary)
