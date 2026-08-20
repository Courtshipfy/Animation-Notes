# Slope Warping 详解

> Slope Warping 做的事一句话就能说清：它不逐脚去「贴地」，而是把整个下半身——脚的参考系、IK 脚目标、骨盆、髋——一起掰到坡面法线上，让「整段步态」跟着地形倾斜。它的聪明之处在于，坡面法线它自己不算，直接复用胶囊碰撞已经做好的地面检测。

## 它解决的问题

动画资产通常按平地录制：整段步态假设地面是平的、脚底平面和地面平行、重心相对脚的高度是固定的。把这段动画直接放到斜坡、楼梯或起伏地形上，问题就来了——脚掌插进坡里或悬空，腿和骨盆还按平地的几何摆，看起来「人没踩在坡上，而是浮在坡上方」。

单脚 Foot Placement（把脚掌旋转到贴合地面、把落点对齐）能解决「脚掌贴合」这一件事，但救不了整体：腿的朝向、骨盆的高度、步态的重心都还是平地的。坡陡一点，脚是贴上去了，膝盖、髋、重心全拧着。

Slope Warping 解决的就是这个「整体适配」问题：把整段步态当作一个整体，绕坡面法线重新定向，让「动画里假设的朝上方向」对齐「真实的坡面朝上方向」。

它明确**不做**三件事，理解了边界才不会误用：

- **不做地面检测**：坡面法线来自胶囊（`CharacterMovementComponent` 的 `CurrentFloor`，即胶囊碰撞已经算好的地面命中），节点自己不发任何射线。胶囊没有地面信息，它就按「平地」处理。
- **不管方向**：它不读移动方向、不改面向。角色朝哪走、下半身朝哪转，是 Orientation Warping 的活，两者各管一摊，互不重叠。
- **不解腿链**：它只改 IK 脚目标的位置/朝向、以及髋的参考朝向，真正把大腿—小腿—脚解算回腿链，是后面 Leg IK 的活。

## 核心思想：复用胶囊的「地面」，把下半身整体重新定向

整个节点可以浓缩成两步：

1. 拿到**真实的坡面法线**——不是自己 trace，而是读 `CharacterMovementComponent` 的 `CurrentFloor`。命中就用 `ImpactNormal`，没命中（空中）就退回 `-GravityDir`，即「按重力方向定义的平地」。
2. 算出动画**当前假设的朝上方向**和真实坡面法线之间的旋转差，把这笔旋转应用到下半身的参考系上。

第 2 步的「动画假设朝上方向」很关键：节点取 `IKFootRootBone`（脚的参考根骨）的 Z 轴当作「动画此刻认为的上」，然后用 `FindBetweenNormals` 求出从「动画上」到「真实坡面法线」的旋转 `DeltaSlopeRotation`，作用到 IK Foot Root、每只脚的 IK 目标，并让骨盆跟着双脚质心一起动。

这样做的效果是：**不是把脚掌一块块掰到坡上，而是把整个下半身当刚体整体旋转到坡面上**。脚掌贴合只是这个整体旋转的副产品，骨盆和腿的几何也随之对齐。它和「只转脚掌」的本质区别就在这里。

> 注意一个容易想当然的地方：Slope Warping **没有「地面法线」输入引脚，也没有「移动方向」输入**。地面法线来自胶囊的地面检测结果（复用，而非取代）；移动方向压根不参与它的计算。想「输入方向」去做的事，属于 Orientation Warping。

## 关键流程：一帧里发生了什么

求解路径可以按顺序拆成几段：

1. **读地面并平滑**。`GetSmoothedFloorInfo` 从移动组件拿到当前地面命中，得到目标法线和一个「地面偏移」——它由三部分合成：胶囊离地的距离、网格与胶囊底部的高差、胶囊半径贴坡带来的偏移。法线在世界空间用弹簧插值，地面偏移在局部空间插值，并被 `MaxStepHeight` 钳制，防止一步迈上过高台阶时步态被硬拽过头。空中或检测到传送时则直接重置、回到平地。

2. **算旋转差**。`FindBetweenNormals(IKFootRoot 的 Z 轴, FloorNormal)` 得到把「动画上」掰到「坡面法线」的四元数。如果几乎就是恒等（本来就在平地或已对齐），跳过旋转，只处理平移偏移。

3. **掰下半身**。有旋转差时：IK Foot Root 旋转并加地面偏移；每只脚的 IK 目标位置被旋过去、再加地面偏移，朝向也做同款旋转；骨盆先加地面偏移，再补一个「质心偏移」——用双脚 IK 目标的质心在旋转前后的位移差，把骨盆钉在双脚质心上方，让整个下半身一起动，而不是骨盆和脚各动各的。

4. **下拉骨盆（`bPullPelvisDown`）**。坡会把两只脚拉开——一只踩高、一只踩低，髋到 IK 脚目标的距离会超过腿的自然长度（过伸）。此时逐脚检查：若「调整后的髋 → IK 脚目标」的距离超过了「原动画里髋 → FK 脚」的长度，就把髋朝脚的方向下拉到刚好等于原腿长，最多迭代 4 次保证每只脚都够得着。这就是「下拉骨盆以容纳脚部高度差」——它解决的不是贴地，而是**腿够不够长**。

5. **留在胶囊里（`bKeepMeshInsideOfCapsule`）**。坡面旋转 + 下拉骨盆，会把身体往坡的侧面带。节点把骨盆相对初始位置的偏移投影到坡面平面上，再把这个「沿坡的侧移」整体减去，保证网格还呆在胶囊内、不穿出碰撞体。

6. **为下游 IK 收拾腿**。骨盆偏移用弹簧平滑后，把髋朝向 IK 脚目标的方向转（`FindBetweenNormals` 从「髋→FK 脚」到「髋→IK 脚」）；同时若 IK 脚目标比 FK 腿长还远，就把 IK 目标沿方向拉回到 FK 腿长——保留动画姿态、防止过伸，好让后面的 Leg IK 有干净的目标可解。

7. **写回骨骼**。把改过的骨骼（IK Foot Root、根骨、骨盆、各 IK 脚）按索引排序后写入 `OutBoneTransforms`。注意根骨只有在存在地面偏移时才会被平移——地面偏移是「整体抬/降」，所以动的是根骨而不是逐脚。

## 和谁配合：它在一段管线里的位置

- **前置：地面检测已在胶囊里发生**。Slope Warping 完全依赖 `CharacterMovementComponent::CurrentFloor`，前提是角色本身在正常 Walking/NavWalking 且有地面命中。它和地面检测是「复用」关系，不是「取代」关系。
- **顺序：Slope Warping → Foot Placement**。先让整体步态对齐坡面，再由 Foot Placement 决定每一脚的精确落点。顺序反了或只用其一，要么脚贴了地但整体拧着，要么整体对了但落点粗糙。
- **后面：Leg IK**。Slope Warping 只改 IK 脚目标，不碰真实腿骨；把目标解算回腿链交给 Leg IK。
- **旁边：Orientation Warping**。Slope Warping 管「坡」（法线），Orientation Warping 管「方向」（移动方向 vs 动画方向）。两者不重叠，常驻 Motion Matching 内层时各自负责自己的那一笔修正。

## 常见问题

- **节点「没反应」**：最常见的原因是拿不到移动组件。`IsValidToEvaluate` 要求 `MyMovementComponent` 非空——它从网格组件的 Owner 上 `Cast<ACharacter>` 再取移动组件。宿主不是 Character、或移动组件不存在，节点直接静默失效。此外还需要 `IKFootRootBone`、`PelvisBone` 配好、且至少一只脚配置有效（IK 脚、FK 脚、髋都能解析出来）。
- **脚下还是穿模/悬空**：先分清是「整体没对齐」还是「落点不准」。整体对齐是 Slope Warping 的活，单脚落点是 Foot Placement 的活；只配一个自然不完整。
- **陡坡上腿被拉得很难看**：检查 `bPullPelvisDown` 和 `MaxStepHeight`。下拉骨盆解决腿过伸，但被 `MaxStepHeight` 钳住的「地面偏移」太大时，整体抬/降会失真。
- **空中/落地瞬间步态弹回**：非着地时目标法线退回「平地」（`-GravityDir`）、目标偏移清零，所以落地瞬间能看到步态从「平」过渡回「坡」，这是弹簧平滑的正常表现，不是 bug。
- **传送后步态突变**：节点检测到 Actor 位移超过网络平滑距离会重置插值器，属于防抖逻辑，属预期行为。
- **`FootSize` 字段没作用**：`FSlopeWarpingFootDefinition::FootSize` 在本版本（UE 5.8）只定义、未在求解路径中读取，属于未使用字段，别指望它改脚掌贴合范围（需在目标引擎版本核对是否后续启用）。

## 参考资料

- [Pose Warping](https://dev.epicgames.com/documentation/en-us/unreal-engine/pose-warping-in-unreal-engine)（官方文档，Slope Warping 属 Pose Warping 一组节点）
- [Game Animation Sample Project](https://dev.epicgames.com/documentation/en-us/unreal-engine/game-animation-sample-project-in-unreal-engine)
- 引擎源码（UE 5.8）：
  - `Engine/Plugins/Animation/AnimationWarping/Source/Runtime/Public/BoneControllers/AnimNode_SlopeWarping.h`
  - `Engine/Plugins/Animation/AnimationWarping/Source/Runtime/Private/BoneControllers/AnimNode_SlopeWarping.cpp`
  - 基类 `Engine/Source/Runtime/AnimGraphRuntime/Public/BoneControllers/AnimNode_SkeletalControlBase.h`
