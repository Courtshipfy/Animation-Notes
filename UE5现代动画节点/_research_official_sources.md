# UE5 现代动画节点：官方资料核对

> 用于拆分专题前核对术语和职责。以下结论以 UE 5.8 官方文档为准；插件、属性名和实验性状态会随小版本变化，落地时应再查看目标引擎版本的节点详情和源码。

## 接触与 IK

### Foot Placement

- **定位**：`FAnimNode_FootPlacement` 是 Animation Warping Runtime 插件公开的动画节点类型；它应单独写成「脚部接触求解」专题，而不要与基础 IK 混为一谈。官方 API 是确认其所属模块和当前公开字段的第一手入口。[`FAnimNode_FootPlacement` API](https://dev.epicgames.com/documentation/en-us/unreal-engine/API/Plugins/AnimationWarpingRuntime/BoneControllers/FAnimNode_FootPlacement)
- **写作边界**：不要把它表述为单纯的射线检测或 Foot Lock。射线/地面信息、接触状态、腿链与骨盆补偿属于同一脚部放置问题的不同输入或求解环节；具体可用输入和属性必须以当前版本 `FAnimNode_FootPlacement` 为准。上面的 API 链接应与所用版本源码交叉核对。

### Leg IK

- **定位**：Leg IK 是 AnimGraph 中的腿链执行层。官方 Pose Warping 教程明确要求在 Orientation Warping 后接 Leg IK，把被修改的 IK 骨骼结果传递回 FK 骨骼；因此它不应被写成「自动检测并锁脚」的完整方案。[Pose Warping：Orientation Warping 设置](https://dev.epicgames.com/documentation/en-us/unreal-engine/pose-warping-in-unreal-engine#orientationwarping)
- **适用场景**：Game Animation Sample 把 Leg IK 描述为在不平地面上生成更自然站姿的开关项。这说明它可作为最终腿部解算，但地面目标、何时保持/释放脚等策略仍需由上游数据决定。[Game Animation Sample：Leg IK](https://dev.epicgames.com/documentation/en-us/unreal-engine/game-animation-sample-project-in-unreal-engine)
- **版本核对入口**：[`FAnimNode_LegIK` API](https://dev.epicgames.com/documentation/en-us/unreal-engine/API/Runtime/AnimGraphRuntime/BoneControllers/FAnimNode_LegIK)

### Full Body IK（FBIK）与 Control Rig

- **定位**：FBIK 建立在 Position Based IK 框架上，官方将它定位为 Control Rig 内的程序化调整工具，例如地面对齐和手臂伸向目标；它不是与 Leg IK 等价的一条普通 AnimGraph 腿节点。[Full-Body IK 概览](https://dev.epicgames.com/documentation/en-us/unreal-engine/control-rig-full-body-ik-in-unreal-engine)
- **工作方式**：在 Control Rig 图中创建 Full Body IK 节点，设定 Root（典型为 hips/pelvis），并为每个控制器添加对应骨骼和 Effector。这正是「多末端共同影响骨盆与全身」的配置依据。[FBIK：创建节点与 Effector](https://dev.epicgames.com/documentation/en-us/unreal-engine/control-rig-full-body-ik-in-unreal-engine#effectorsetup)
- **调节原则**：每根受影响骨骼可设置 Position/Rotation Stiffness；该值控制骨骼受 Effector 影响的程度。专题应把它解释为分配身体响应，而非盲目提高迭代或权重。[FBIK：Stiffness](https://dev.epicgames.com/documentation/en-us/unreal-engine/control-rig-full-body-ik-in-unreal-engine#rotationandpositionstiffness)

## 姿势与根运动修正

### Orientation Warping

- **作用对象**：Orientation Warping 对运动中的动画姿势施加方向补偿，隔离并扭转腿部 IK 骨骼来对齐持续变化的根运动移动方向，同时用脊柱扭转维持朝向；它修的是**当前姿势的方向残差**，不是驱动 Actor 转向。[官方定义与属性](https://dev.epicgames.com/documentation/en-us/unreal-engine/pose-warping-in-unreal-engine#orientationwarping)
- **实现要点**：节点需要配置 IK Foot Root、两只 IK Foot 和至少一根 Spine；Graph 模式可输入动态 Locomotion Angle。官方示例也说明它后面需要 Leg IK。[官方设置流程](https://dev.epicgames.com/documentation/en-us/unreal-engine/pose-warping-in-unreal-engine#orientationwarping)

### Stride / Slope Warping

- **Stride Warping**：依据胶囊移动速度和相关骨骼动态改变双脚间距，以匹配动画步幅与角色速度；它调整的是空间步幅，不等于只改播放速率。[Pose Warping：Stride Warping](https://dev.epicgames.com/documentation/en-us/unreal-engine/pose-warping-in-unreal-engine#stridewarping)
- **Slope Warping**：帮助把脚部位置适配地面法线，使斜坡和台阶上的移动过渡更平滑；节点也提供向下拉骨盆以容纳新脚位的选项。它可作为「整体坡面适配」专题，与单脚接触放置区分。[Pose Warping：Slope Warping](https://dev.epicgames.com/documentation/en-us/unreal-engine/pose-warping-in-unreal-engine#slopewarping)

### Motion Warping

- **定位**：Motion Warping 动态调整角色的 Root Motion，使其对齐目标；官方流程包含角色蓝图逻辑、Montage 中的 Motion Warping Window，以及命名目标位置的关联。因此它应归入「根运动到场景目标」专题，不应承担连续移动的脚部贴地职责。[Motion Warping 概览](https://dev.epicgames.com/documentation/en-us/unreal-engine/motion-warping-in-unreal-engine)
- **最小配置**：需要在角色上添加 Motion Warping Component；Warp Target Name 与蓝图中的 Add or Update Warp Target 对应；用于 Warp 的动画必须启用 Root Motion。[组件、目标与前置条件](https://dev.epicgames.com/documentation/en-us/unreal-engine/motion-warping-in-unreal-engine#motionwarpingcomponent)
- **时间边界**：Skew Warp 在指定 Warping Window 的结束时使 Root Motion 匹配目标位置和旋转，因而专题必须强调 Notify State 的开始/结束时间，而非把 Warp 当成全动画恒定修正。[Root Motion Modifier / Skew Warp](https://dev.epicgames.com/documentation/en-us/unreal-engine/motion-warping-in-unreal-engine)

### Distance Matching

- **定位**：Distance Matching 按角色与目标的距离变量驱动 Animation Sequence，而不是按线性时间播放；动画曲线把姿势与距离关联，运行时在曲线上查找相应姿势。[Distance Matching 概览](https://dev.epicgames.com/documentation/en-us/unreal-engine/distance-matching-in-unreal-engine)
- **实现链路**：官方落地示例使用 Sequence Evaluator、动画 Distance Curve 和动态距离（示例为离地剩余距离）选择落地姿势；这是一套「曲线 + Evaluator + Anim Node Function」工作流，而非独立的输入 Pose/输出 Pose 修正节点。[Distance Matching 工作流](https://dev.epicgames.com/documentation/en-us/unreal-engine/distance-matching-in-unreal-engine#distancematchingworkflow)
- **资源前提**：运行时读取距离曲线需要 Uniform Indexable Animation Compression Setting，专题应将其列为配置核对项。[官方压缩设置说明](https://dev.epicgames.com/documentation/en-us/unreal-engine/distance-matching-in-unreal-engine)

## 姿势检索与动作匹配

### Pose Search 与 Motion Matching

- **定位**：Motion Matching 是 Pose Search 插件中的查询式姿势选择系统；它可动态替代部分 State Machine 或 Blend Space，但本身解决的是「从候选数据中选哪个姿势」，并不自动完成地面接触或 IK。[Motion Matching 概览](https://dev.epicgames.com/documentation/en-us/unreal-engine/motion-matching-in-unreal-engine)
- **数据流**：Motion Matching 节点通过 Schema 查询角色通道（如骨骼位置、速度），再从 Database 资产的动画数据中选姿势。Pose Search Schema 保存配置与查询设置，并连接数据库、查询系统和节点。[Schema 与查询机制](https://dev.epicgames.com/documentation/en-us/unreal-engine/motion-matching-in-unreal-engine#createaposesearchschemaasset)
- **性能取舍**：更多 Channel / Sample 会带来更细的检索特征，但增加内存与运行成本；这是配置专题应明确记录的质量—成本关系。[Schema Channel 参考](https://dev.epicgames.com/documentation/en-us/unreal-engine/motion-matching-in-unreal-engine#posesearchschemaassetreference)
- **Pose History 的职责**：官方 Game Animation Sample 明确将 Pose History 作为 Motion Matching 所需的第二个 AnimGraph 节点：它保存过往姿势，并持有或生成用于匹配的 Trajectory。它不能与只为避免重复计算的 Cached Pose 混同。[Game Animation Sample：Pose History](https://dev.epicgames.com/documentation/en-us/unreal-engine/game-animation-sample-project-in-unreal-engine)

## 对应的专题边界

- `01-接触与IK.md`：Foot Placement、Leg IK、FBIK / Control Rig；按「接触目标 → 腿链 → 全身约束」解释职责。
- `02-Pose-Warping.md`：Orientation、Stride、Slope Warping；说明三者的输入、作用空间与覆盖范围。
- `03-根运动与根骨修正.md`：Motion Warping 的 Root Motion、Warp Window、Warp Target；Offset Root Bone 另作内部 Mesh 偏差处理。
- `04-Distance-Matching.md`：Distance Curve、Sequence Evaluator、节点函数和压缩设置。
- `05-查询数据与资源选择.md`：Pose History、Trajectory、Chooser、Proxy Table；Motion Matching 的数据库细节仍由 `MotionMatching/` 目录承担。
- `06-动态姿势过渡.md`：Blend Stack、Inertialization、Dead Blending，只讨论姿势切换而不讨论姿势选择。

## 官方资料索引

- [Pose Warping](https://dev.epicgames.com/documentation/en-us/unreal-engine/pose-warping-in-unreal-engine)
- [Motion Warping](https://dev.epicgames.com/documentation/en-us/unreal-engine/motion-warping-in-unreal-engine)
- [Distance Matching](https://dev.epicgames.com/documentation/en-us/unreal-engine/distance-matching-in-unreal-engine)
- [Full-Body IK](https://dev.epicgames.com/documentation/en-us/unreal-engine/control-rig-full-body-ik-in-unreal-engine)
- [Motion Matching](https://dev.epicgames.com/documentation/en-us/unreal-engine/motion-matching-in-unreal-engine)
- [Game Animation Sample Project](https://dev.epicgames.com/documentation/en-us/unreal-engine/game-animation-sample-project-in-unreal-engine)
