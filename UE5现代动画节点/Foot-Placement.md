# Foot Placement（脚部放置）详解

> Foot Placement 做的事一句话就能说清：把「该贴地的脚」连续地锁在一张被平滑过的地面平面上，再反过来求骨盆该往哪挪，让每条腿都不被拉长、也不被压弯。它的关键不是「每条腿打一条射线 + Two Bone IK」，而是维护一个跨帧的接触状态——落地、锁定、抬脚是一个有记忆的连续过程，而不是每帧重新猜一次脚该在哪。

## 它解决的问题

动画里的脚是相对根骨运动的。哪怕某一帧脚掌看起来踩在地上，只要胶囊继续移动、旋转，或者多个动画正在混合，脚骨的世界位置就会跟着身体漂移——表现出来就是脚底打滑。同时，动画是在「水平参考平面」上录的：下坡时脚悬空、上坡时脚陷进地面、站在斜面上脚掌还保持水平。

Stride Warping 解决「这一步该跨多远」，Foot Placement 解决的是更底层的问题：**脚和地面之间到底是一种什么关系**——脚贴不贴地、贴在哪、朝哪，以及为了贴住地面，骨盆要不要跟着让一步。

它明确**不做**三件事，理解了边界才不会误用：

- **不选动画**：它不能把一段平地走路变成上楼梯走路，只在现有姿势上修正接触。动作选择是上游（Motion Matching / 状态机）的活。
- **不算 Two Bone IK**：节点只输出骨盆变换和每只脚的 IK 脚目标，从来不碰大腿—小腿—脚踝的骨骼链。把腿真正解算到目标是下游 Leg IK 的活。
- **不驱动 Actor 移动**：它只改骨骼姿势，胶囊该怎么走还怎么走。

## 核心思想：接触是「跨帧状态」，不是一帧的射线

一条射线 + Two Bone IK 的思路是「无状态」的：每帧从脚往下打条射线，命中点就是脚该去的地方，再用 IK 把腿拽上去。它的问题在于没有记忆——地面一旦抖动、脚速刚过阈值、动画混合突变，命中点就跳，脚就 Pop；而且每条腿各自解算，谁也不管骨盆，腿很容易被拉到极限。

Foot Placement 的做法反过来了：它给每条腿维护一份运行时状态 `FLegRuntimeData`，跨帧记住四样东西：

- **种植平面** `PlantPlaneRS`：脚当前踩在的「地面平面」，存在根空间，帧与帧之间延续并平滑。
- **锁定/未锁定脚变换**：`AlignedFootTransform` 与 `UnalignedFootTransform`（各存一份世界空间和根空间）。
- **接触状态机**：`PlantType`（Planted / Unplanted / Replanted）、上一帧类型、扭转修正量、`bCanReachTarget` 等。
- **一串弹簧状态**：脚锁定偏移、种植平面高度、地面法线，各自用弹簧插值平滑。

于是每一帧的求解，都不是「从零猜脚在哪」，而是「在上一帧接触状态的基础上，用本帧姿势微调」。射线检测（sphere sweep）只是**其中一路输入**，用来更新那张平滑的地面平面；脚最终去哪，是「上一帧锁定变换 + 平滑平面 + 状态机」共同决定的。

| | 射线 + Two Bone IK | Foot Placement |
|---|---|---|
| 接触记忆 | 无，每帧重新算 | 有：平面、锁定偏移、状态机跨帧延续 |
| 求解对象 | 单条腿 | 全部腿 + 骨盆，耦合求解 |
| 脚去哪 | 射线命中点 | 平滑种植平面上的锁定变换 |
| 骨盆 | 不动 | 主动反推偏移 |

## Plant / Swing 状态怎么判定与切换

接触状态不是「脚速低于阈值就锁、高了就放」那么简单。它分成两层：**想不想锁**和**实际锁没锁**。

### 想不想锁（WantsToPlant）

四条件同时成立才算「想锁」：

1. `LockType != Unlocked`，且当前帧的 `LockAlpha > 0`；
2. 脚到地面平面的距离 `DistanceToPlant < DistanceToGround`（够近）；
3. 脚速 `Speed < SpeedThreshold`（够慢）。

`LockAlpha` 默认由 `DisableLockCurveName` 曲线控制（`1 - 曲线值`，没配曲线就是 1），可以用动画曲线**精确地**在某几帧关掉锁定。脚速的来源由 `PlantSpeedMode` 决定：`Manual` 读每条腿的 `SpeedCurveName` 曲线（默认），`Graph` 则用根运动属性流加上 ball 骨自身的位移来算。

另一个量是**对齐 alpha**（`GetAlignmentAlpha`）：它把脚速从 `[UnalignmentSpeedThreshold, SpeedThreshold]` 映射到 `[0, 1]`——0 表示脚在空中、完全不对齐，1 表示已锁住、完全贴地。它不参与「锁不锁」的判断，而是用来在锁与不锁之间做连续过渡，避免落地瞬间硬切。

### 实际锁没锁（DeterminePlantType）

`WantsToPlant` 只是「意愿」，真正的状态机用**迟滞**（hysteresis）来避免脚在阈值边界反复抓放：

- **已锁**（上一帧 Planted/Replanted）：只要脚相对 FK 脚的位置偏移没超过 `UnplantRadius`、扭转角没超过 `UnplantAngle`，就**沿用上一帧的种植状态**（继续锁）。这是关键——一旦锁上，就不会因为脚速轻微波动而立刻松掉。
- **上一帧想锁但没锁成**（正在找机会再种）：只有当脚回到旧锁点附近——偏移落在 `ReplantRadius`（= `UnplantRadius × ReplantRadiusRatio`）、扭转落在 `ReplantAngle` 内——才进入 `Replanted`。
- **上一帧根本不想锁、这帧想锁**：直接进入 `Planted`，锚定当前位置。

两套阈值的意义在于：**解锁比解锁「难」，再种比新种「难」**。`ReplantRadiusRatio`、`ReplantAngleRatio` 决定了这个迟滞环的宽窄；把它们都设为 1 时，就没有「再种」这一说了——偏移直接 clamp 到半径内，脚变成「钉住滑动」而不是「松了再抓」。

## 落脚点与朝向怎么确定

一旦确定「这帧在锁」，落点与朝向分两步定。

### 锁定锚点：LockType

`LockType` 决定「锁」的锚点语义，算出未贴地的锁定变换：

- `PivotAroundBall`：绕脚掌 ball 骨旋转，让 ball 保持原地，脚掌以后跟抬起的姿态转动（最常用，保留脚掌滚动感）。
- `PivotAroundAnkle`：只锁位置（沿用上一帧脚踝位置），朝向可以变。
- `LockRotation`：位置和朝向都沿用上一帧（适合大型/机械生物，脚完全钉死）。

这得到一个「未对齐」的锁定变换，它表达的是「脚在动画里想怎么动、但被上一帧锚点拉住了多少」。

### 贴地：种植平面 + 地面朝向对齐

脚最终朝向由 `AlignPlantToGround` 完成。它做三件事：

1. **保持脚与「IK 脚根平面」的竖直距离不变**，把脚沿逼近方向（重力反方向）投影到种植平面上——保证脚不会因为平面换了就忽高忽低。
2. **把脚掌朝向对齐到种植平面的法线**（贴坡）。
3. 把对齐产生的旋转拆成 **swing（摆动）与 twist（扭转）**：swing 保留（脚掌顺着坡面起伏），twist 按 `AnkleTwistReduction` 打折——默认不保留全部扭转，避免脚掌为了贴地而绕脚踝轴拧过头。

种植平面本身不是每帧从射线直接拿的：`UpdatePlantingPlaneInterpolation` 会用一个高度弹簧 + 一个法线弹簧把它平滑（`bEnableFloorInterpolation` 控制），存在根空间。这样地面几何即便有台阶、凸起，脚也是「慢慢贴过去」而不是「瞬间弹上去」。`MaxGroundPenetration` 还允许在插值期间允许一点穿地，防止平滑跟不上时脚悬空。

整个过程再用 `LockAlpha` 把「未锁」和「锁定+贴地」混合，`LockAlpha` 归零就自动解锁。

## 骨盆补偿：为什么存在、何时介入

这是 Foot Placement 和「只做脚」的方案最大的分水岭。

**为什么必须动骨盆**：脚一旦被锁到地面平面上，骨盆若不动，腿长就被强行改变——地面下陷时腿被拉直，脚被锁住向上推时腿被压弯。骨盆补偿就是把「脚贴地的代价」分摊到骨盆上，让腿尽量保持在自然伸展范围内。

`SolvePelvis` 分两步：

1. **水平重平衡**：按 `HorizontalRebalancingWeight`，取所有脚偏移的水平均值，把骨盆往被锁脚偏移的反方向挪一点，重新分配重心。
2. **求竖直偏移**：对每条腿，用「球—线相交」（参考 Rune Skovbo Johansen 论文第 7.4.2 节）算出骨盆相对该脚的可达范围——`DesiredExtension`（保持动画里腿长）、`MaxExtension`（允许伸展上限，由 `MaxExtensionRatio` 决定）、`MinExtension`（允许压缩下限，由 `MinExtensionRatio` 决定）。然后把所有腿的 Min/Avg/Max 综合起来求一个折中 Z 偏移，并 clamp 到「不过度伸展、也不过压缩」的区间。

它**何时介入、何时退出**由几层控制：

- 结果是「目标偏移」，还要过一道弹簧插值（`bEnableInterpolation`），再被 `MaxOffset` 限幅——所以骨盆不会硬跳。
- `bDisablePelvisOffsetInAir` 默认开着：离地时骨盆偏移被关掉，脚回到源姿势，避免空中姿态被地面逻辑拉歪。
- `ActorMovementCompensationMode` 处理「胶囊自己在竖直方向动」的情况：`ComponentSpace` 完全跟随胶囊；`WorldSpace` 无视胶囊竖直位移、只让弹簧消化；`SuddenMotionOnly`（默认）只在胶囊突然跳变（如大步台阶）时补偿。三者应对的是「骨盆该跟随平滑地面，还是该钉在世界里」的取舍。
- `HeelLiftRatio` 决定「先抬脚跟还是先降骨盆」：值越大越优先用抬脚跟消化差异，而不是把骨盆往下压。
- `DisablePelvisCurveName` 可以用曲线把骨盆偏移整体淡出。

> 一个版本核对点：`PelvisHeightMode` 在 UE 5.8 源码里**只声明、未接线**——`SolvePelvis` 始终遍历全部腿，三个枚举值（AllLegs / AllPlantedFeet / FrontPlantedFeetUphill…）目前不改变行为。依赖它做「只按支撑脚定骨盆」需在目标引擎版本核对。

## 与旧 Foot Lock / Foot Offset / Pelvis Offset 的职责边界

这套节点把过去分散在三个环节里的活，合并进一个**耦合的连续求解**。以本仓库已记录的 ALS 旧链路（Foot Lock → Foot Offset → Pelvis Offset，最后 Two Bone IK）为对照：

| 旧环节 | 旧职责 | Foot Placement 里对应的阶段 | 差异 |
|---|---|---|---|
| Foot Lock | 支撑脚防水平滑动，锁世界锚点 | `DeterminePlantType` + `LockType` + `UnalignedFootOffset` | 锁的是「平面 + 变换」而非单个世界点；用程序化速度/距离 + 迟滞判定，而不是只靠曲线「满权重才捕获」；支持绕 ball/脚踝枢轴 |
| Foot Offset | 射线找地面、贴坡、限幅 | `UpdatePlantingPlaneInterpolation` + `AlignPlantToGround` | 用 sphere sweep + 平滑种植平面，swing/twist 拆分与 `AnkleTwistReduction` 取代小腿空间三轴限幅 |
| Pelvis Offset | 地面压脚后下拉骨盆保腿长 | `SolvePelvis` + 弹簧插值 | 与脚在同一节点内**耦合求解**：骨盆以对齐后的脚为输入，脚再以骨盆为输入（`FinalizeFootAlignment`），不再需要「用 `Max()` 硬拼两条高度链的一致性」 |

一句话：**旧链路是三个各自独立的修正器串行工作，靠约定保持不打架；Foot Placement 把「锁、贴、挪骨盆」放进同一个有状态的求解器，让它们共享同一份接触状态。** 而真正把腿链（大腿—小腿—脚踝）解算到目标的 Two Bone IK，两种方案都**外放**给了下游 Leg IK——Foot Placement 自己从头到尾不碰这条链。

## 它不碰什么、和谁配合

- **后面必须接 Leg IK**。Foot Placement 只把骨盆和每只脚的 IK 脚骨写出去，从不解算腿链；没有 Leg IK（或 Control Rig 里消费 IK 脚目标的等价物），这些目标就落不到真实腿姿上。
- **尽量放在 Motion Matching 内层**，读「接近最终状态」的腿姿势。它之后若还有节点改动骨盆或腿（比如再叠一层 IK 或 Spine 调整），脚可能被再次拉离地面。
- **依赖上游提供脚速**：`Manual` 模式要脚速曲线，`Graph` 模式要根运动属性流（通常来自 Motion Matching 或开了 Root Motion 的 Sequence Player）。脚速是种植判定的地基。
- **`IKFootRootBone`（`ik_foot_root`）是整套对齐的基准**：它定义了「IK 脚根平面」，脚到地面的距离、贴地时保持的竖直距离，都以它为参照。配错它，种植判定和对齐都会偏移。

## 常见问题

- **节点「没反应」**：先查骨骼配置。`IsValidToEvaluate` 要求每条腿的 Hip / FK / IK / Ball 四个索引、以及骨盆的 FK / IK 索引全部有效，缺一个整个节点静默不跑。
- **脚一直不锁**：逐项核对 `WantsToPlant` 的四条件——`LockType` 不是 Unlocked、`LockAlpha` 不为 0、`DistanceToGround` 够大、`SpeedThreshold` 够高；Manual 模式下尤其检查 `SpeedCurveName` 是否真的写在了动画里（曲线缺失时脚速回退成阈值，判定会退化）。
- **落地/离地瞬间脚 Pop**：优先看种植平面插值——`bEnableFloorInterpolation` 是否被关、地面高度/法线弹簧刚度是否太小；再看 `LockAlpha` 的对齐 alpha 过渡是否留了时间。
- **坡面上脚掌被强行回正**：查 `AnkleTwistReduction` 和 swing/twist 拆分。它默认只保留部分扭转，动画本身若带明显脚踝扭转，需要调这个值，而不是怀疑贴坡失效。
- **腿被拉直或压弯**：这是伸展约束和骨盆求解共同作用的结果——查 `MaxExtensionRatio` / `MinExtensionRatio` 与骨盆 `MaxOffset`。设得过大，腿会明显拉长/压弯。
- **传送后脚锁残留**：节点有传送检测（根骨世界位移超过组件 `TeleportDistanceThreshold` 就 `ResetRuntimeData`），但阈值来自组件，位移较小的场景传送可能不触发。
- **脚下微抖或慢半拍**：弹簧参数问题——刚度高 + 阻尼低会抖，刚度低或阻尼高则跟手慢；地面、锁定偏移、骨盆三套弹簧是独立参数，分开调。

## 参考资料

- [Pose Warping](https://dev.epicgames.com/documentation/en-us/unreal-engine/pose-warping-in-unreal-engine)（Foot Placement 所属的 Pose Warping 功能组官方文档）
- [FAnimNode_FootPlacement API 参考](https://dev.epicgames.com/documentation/en-us/unreal-engine/API/Plugins/AnimationWarpingRuntime/FAnimNode_FootPlacement)
- [Game Animation Sample Project](https://dev.epicgames.com/documentation/en-us/unreal-engine/game-animation-sample-project-in-unreal-engine)
- 引擎源码（UE 5.8）：`Engine/Plugins/Animation/AnimationWarping/Source/Runtime/`（`FAnimNode_FootPlacement` 位于 `Public/BoneControllers/AnimNode_FootPlacement.h` 与 `Private/BoneControllers/AnimNode_FootPlacement.cpp`，运行时状态与设置结构同在同目录头文件）
