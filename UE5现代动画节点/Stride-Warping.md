# Stride Warping 详解

> Stride Warping 做的事一句话就能说清：它改的是「空间步幅」而不是「时间节奏」。不加快动画播放，而是按「实际移动速度 ÷ 动画参考速度」的比例，把每只脚沿移动方向拉长或收窄，让每一步跨过的距离跟得上真实位移，从而消除脚底打滑。

## 它解决的问题

动画资产只烘焙了一种速度。一段「中速跑」循环里，脚的迈步距离、踩地节奏、根运动位移，全都锁死在录制时的速度上。现在角色想跑快一点——胶囊速度上去了，但动画还是原来那个步幅：脚每步只往前迈那么远，身体却已经滑出去更多，脚底和地面之间产生相对滑动，看起来像在冰面上原地蹬。

要消除这个「步幅和速度不匹配」，本质上有两根杠杆：

- **改时间（Play Rate）**：把动画播快点/播慢点，让同样的步幅在更短/更长的时间里走完。这改的是**节奏**——步频、踩地时机、整个步态的「快慢感」都变了，跑快了容易显得动作被快进、不自然。
- **改空间（Stride Warping）**：保持动画自己的节奏不变，只把每步的**跨距**拉长或收窄。步频不变，脚每步走得更远，步态的形状被等比拉伸。

Stride Warping 是第二根杠杆。它特别适合围绕某个参考速度做小幅修正（±20% 上下），因为这时动作的「踩地节奏感」被完整保留，只微调跨距，观感最自然。

它也明确**不做**三件事，理解边界才不会误用：

- **不选动画**：它不能把走路变成跑步——步频、抬腿高度、腾空这些「步态特征」它改不了，只能拉长现有的腿。速度差得太多，还是得换资产。
- **不改节奏**：动画的播放时长、踩地时机原封不动。想让整个动作快点，那是 Play Rate 的活，两者互补而非互斥。
- **不解算腿链**：它只改 IK 脚的目标位置（和骨盆、大腿的引导变换），真正把大腿—小腿—脚解算回去对齐目标，是后面 Leg IK 的活。

## 核心思想：一个比例，摊到下肢几何上

整个节点的求解浓缩成一句话：

```text
步幅比例 = 实际移动速度 / 动画参考（根运动）速度
```

- 比例 **> 1**：角色比动画跑得快 → 步幅拉长，每步多跨一点；
- 比例 **< 1**：角色比动画慢 → 步幅收窄；
- 比例 **= 1**：恰好是动画的参考速度 → 不动，动画原样输出。

这个「参考速度」就是动画自己的根运动速度——动画本帧实际往前位移了多远，除以动画图的 delta time。它是这段动画「天然该有的速度」，移动速度与它的比值，就是我们需要把步幅放大的倍数。

把比例求出来之后，剩下的全是**下肢几何的活**：把每只 IK 脚沿步幅方向、以髋部正下方为轴心缩放；腿长不够时把骨盆往下拉；再转一转大腿引导下游 IK 收敛。没有一个动作动到「播放时间」。

## 步幅比例怎么算

### Manual：直接给比例

最简单的情况——你在引脚上直接给 `StrideScale` 和 `StrideDirection`（组件空间，默认正前方）。节点就按这个比例、沿这个方向掰。适合你已经有了一套自己的速度换算逻辑、或者想手工微调的场景。

### Graph：从根运动速度和移动速度反推

现代 Locomotion（尤其 Motion Matching）默认用 Graph 模式。此时 `StrideDirection` 和 `StrideScale` 两个引脚被**忽略**，节点改从属性流里取：

1. 从上游累计的根运动属性里提取本帧位移（`RootMotionTransformDelta`），它的方向就是动画本帧的前进方向，归一化后作为步幅方向（取不到方向时沿用上一帧的方向）。
2. 用这个位移除以动画图 delta time，得到**根运动速度** `RootMotionSpeed`。
3. 用你喂进来的 `LocomotionSpeed`（胶囊/物理的实际速度）除以它，得到步幅比例：

```text
StrideScale = LocomotionSpeed / RootMotionSpeed
```

**Graph 模式有一个关键副作用**：算出比例后，它不只掰姿势，还会把上游传下来的那笔根运动位移**同样乘上这个比例再写回属性流**。这意味着胶囊的实际位移也被一起缩放——角色真的每步多走这么多，和拉长后的步幅严格一致。换句话说，在 Graph 模式里，`LocomotionSpeed` 成了「事实标准」：姿势和胶囊位移都向它对齐。

有两个容易踩的坑值得单独讲：

1. **没有根运动就不掰**。起步、停止、待机这些帧没有根运动位移，也就没有「参考速度」可言。节点设了速度门槛：根运动速度低于阈值（或接近 0）时，比例直接归 1，靠平滑慢慢回落。这就是为什么建议在这种情况下打开 StrideScaleModifier 的插值——否则比例会从 1.x 硬跳回 1，脚步 Pop 一下。
2. **`LocomotionSpeed` 的单位要对**。源码注释专门强调：这个速度要**相对动画图的 delta time**。因为根运动速度是用动画图的 delta time 算的，两边单位不一致，比值就全错了。别想当然地塞进一个「游戏世界速度」而不管时间基准。

不管哪种模式，最后都会过一次 `StrideScaleModifier`（可选钳制 + 插值），把比例平滑地追到目标值，避免速度突变时步幅 Pop。

## 它具体动哪些骨骼

节点配置里，每只脚要指定三根骨头：**IK 脚**（被驱动的那只，真正写位置）、**FK 脚**（原动画里的脚，只读、当参照）、**大腿**（腿链到骨盆前的那一节）。外加两个全局骨骼：**骨盆**和 **IK 脚根**（步幅/地面/重力方向的参考空间）。求解时按这个顺序动它们：

### 1. 沿步幅轴缩放 IK 脚

先求一个「缩放原点」。思路是让缩放轴心落在**髋部正下方**，而不是世界原点，这样步幅是相对身体拉长：

- 把大腿骨位置沿重力方向投影到「脚所在高度的水平面」上，得到脚下正上方的参考点；
- 以它为锚，把 IK 脚位置投影到「垂直于步幅方向的平面」上，得到缩放原点 `ScaleOrigin`——此时「脚 − ScaleOrigin」恰好是**纯步幅方向**的分量；
- 把这个分量乘上比例再加回去：

```text
WarpedLocation = ScaleOrigin + (IKFootLocation - ScaleOrigin) × StrideScale
```

效果：脚**只沿步幅方向前后移动**，高度和左右位置不变；比例大于 1 脚往前伸，小于 1 脚往回缩。

### 2. 骨盆下拉（保持脚贴地、防过伸）

腿是有固定长度的。步幅拉长后，脚被推到比大腿+小腿能伸到的地方更远，要么膝盖被拉断，要么脚离地。解决方式是**把骨盆往下拉**：髋部降低，同样长度的腿就能够到更远的脚，同时保持脚踩在地面上。

节点先记下每只 FK 脚到骨盆的参考距离（即原动画里的真实腿长），再把拉长后的 IK 脚位置交给 `PelvisIKFootSolver` 迭代求解骨盆该降多少。这个求解器用弹簧插值平滑，带一个 alpha 保留一部分原始骨盆起伏（不然骨盆被拉平，丢了原有的上下律动），并设了最大下拉距离防止极端比例下骨盆被拖到地上。

### 3. 大腿旋转补偿 + FK 腿长钳制

IK 脚和骨盆都动过之后，腿的朝向已经和原动画不一样了。节点会**预先转一下大腿**，让它从「大腿→FK 脚」的方向转到「新大腿→新 IK 脚」的方向——这一步相当于给下游 Leg IK 一个更好的初值，帮它收敛，也尽量保住整条腿原本的形状。

接着按需做一步**钳制**：如果拉长后的 IK 脚到新大腿的距离超过了 FK 腿的原始长度，就把脚沿这条方向收回腿长以内，杜绝膝盖过度拉伸。

### 4. 方向投影到地面

步幅方向在使用前会先投影到地面平面（`bOrientStrideDirectionUsingFloorNormal`）。在斜坡上这很重要：投影后脚是「贴着坡面」滑，而不是沿世界水平方向戳进坡里或翘起来。地面法线默认取世界向上，有坡面时配成真实地面法线才准。

## 和谁配合、边界在哪

- **后面必须接 Leg IK**。Stride Warping 只改 IK 脚目标（以及骨盆、大腿的引导变换），从不碰大腿—小腿—脚的实际骨骼链。要把被推出去的 IK 脚落实成真实腿姿，得靠下游 Leg IK 解算。IK 脚和 FK 脚的「双链」搭法，正是为了让 Leg IK 有参照可对齐。
- **常驻 Motion Matching 内层，和 Orientation Warping 同级配合**。它一般放在 Blend Stack Graph 里，对每个候选姿势统一做步幅修正；Orientation Warping 管「方向」，Stride Warping 管「步幅」，一个掰方向、一个拉跨距，各司其职。
- **和 Stride Blend（ALS 方案）属于同一层**。两者解决的都是同一个「步幅 vs 速度不匹配」问题，都输出一个「速度比例 → 步幅缩放」的系数，在动画的基线速度下这个系数都应当接近 1（不动）。区别在实现载体：ALS 把 Stride Blend 做在项目侧的 AnimBP 里，UE 5 则把它固化成引擎内置节点。
- **极端速度仍要换动画**。比例是「线性拉跨距」，但步态特征（步频、抬腿高度、是否腾空）不会跟着变。走路动画拉到 1.5 倍以上就会像「大步流星却还是走路节奏」，再往后直接散架。所以它是**围绕参考速度的小幅修正**，速度跨档了还是得靠 Blend Space / Motion Matching 换资产——这就是为什么它常驻 Motion Matching 内层：候选已经按速度挑得差不多了，它只负责把残余的几成误差抹平。
- **和 Play Rate 互补**。Play Rate 改「多快走完一步」，Stride Warping 改「一步走多远」。修速度优先用 Stride Warping（保节奏），节奏本身不对了才动 Play Rate。

## 常见问题

- **节点「没反应」**：先查配置是否完整——骨盆、IK 脚根、每只脚的 IK/FK/大腿三根骨头缺一个，`IsValidToEvaluate` 就直接返回 false，节点静默失效。这不是比例算错，是骨骼没配齐。
- **Graph 模式不动，Manual 模式正常**：Graph 模式的前提是上游真的写了根运动属性。普通 Sequence Player（尤其没开 Root Motion）取不到，节点会 early-return；确认上游是 Motion Matching / 带根运动的源，或者检查 `bDisableIfMissingRootMotion` 是否拦住了。
- **起步/停止瞬间脚步 Pop**：这是无根运动帧在作怪——比例被阈值归 1 了，但没平滑就硬跳。打开 `StrideScaleModifier` 的插值，让比例慢慢回落。
- **脚被拉断 / 膝盖过度伸直**：骨盆下拉或 FK 腿长钳制被关掉（`bCompensateIKUsingFKThighRotation`、`bClampIKUsingFKLimits`），或骨盆求解器的最大距离设得太小。极端比例下这些保护是必需的。
- **Graph 模式角色位移也变了，感觉「多动了一层」**：这是 Graph 模式的**正常副作用**——它把根运动位移乘了同样的比例写回，胶囊位移本就应该跟着变。如果别处也做了一次速度补偿，就会叠加成双重缩放，检查链路里是否重复。
- **坡面上脚方向怪**：确认 `FloorNormalDirection` 配的是真实地面法线（而非默认世界向上），并打开 `bOrientStrideDirectionUsingFloorNormal`。
- **`LocomotionSpeed` 喂了但比值总不对**：核对时间基准——它要相对动画图 delta time，和根运动速度同单位。

## 参考资料

- [Pose Warping](https://dev.epicgames.com/documentation/en-us/unreal-engine/pose-warping-in-unreal-engine)
- [FAnimNode_StrideWarping（UE 5.8 API）](https://dev.epicgames.com/documentation/unreal-engine/API/Plugins/AnimationWarpingRuntime/FAnimNode_StrideWarping)
- [Game Animation Sample Project](https://dev.epicgames.com/documentation/en-us/unreal-engine/game-animation-sample-project-in-unreal-engine)
- 引擎源码（UE 5.8）：`Engine/Plugins/Animation/AnimationWarping/Source/Runtime/`（`FAnimNode_StrideWarping` 位于 `BoneControllers/AnimNode_StrideWarping.h/.cpp`；共享的求解器与类型在 `Engine/Source/Runtime/AnimGraphRuntime/Public/BoneControllers/BoneControllerSolvers.h`、`BoneControllerTypes.h`；步幅比例钳制/插值在 `Engine/Source/Runtime/Engine/Classes/Animation/InputScaleBias.h` 的 `FInputClampConstants`）
