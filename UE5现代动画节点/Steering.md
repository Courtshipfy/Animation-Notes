# Steering 详解（根运动轨迹引导）

> Steering 一句话就能说清：让「已经选好的动画」在播放途中，把移动轨迹逐步拐向一个目标朝向——它不像 Orientation Warping 那样掰腿和脊柱的姿势，而是只改写这一帧的根运动旋转，先看「这段动画未来会转多少」做个预测缩放，再用一个阻尼弹簧把剩下的偏差补掉。

## 它解决的问题

Motion Matching 选出下一段动画后，这段动画自带一条固定的根运动轨迹——朝哪走、转多少，都锁在资产里。但玩家的输入是连续的意图（摇杆正在缓缓向右偏），数据库里却只有离散的动画。挑出来的动画转向幅度，几乎永远不会刚好等于你这一刻想转的量：要么转少了、要么转过头。

Steering 就是来补这个「轨迹残差」的：它先预测当前动画接下来会转多少，再和你给的 `TargetOrientation` 一比，用「缩放 + 修正」让实际轨迹逐步靠向目标。

它明确**不做**三件事，理解了边界才不会误用：

- **不选动画**：它假设动画已经选好了，只修正轨迹，不能把向前跑变成横移跑。
- **不动骨骼姿势**：整个求值不产出任何骨骼变换（`OutBoneTransforms` 始终为空），腿、脊柱、根骨的局部旋转一概不碰。
- **不改位移大小**：只改这一帧根运动的旋转（yaw），平移量原样保留。距离控制是 Distance Matching / Stride Warping 的活。

## 核心思想：预测缩放 + 弹簧补差

头文件里的一句话就是全部思路：给根运动属性「加一个程序化增量」，分两种技术、效果叠加——**1) 缩放动画自身的根运动；2) 在考虑过（可能已被缩放的）预期根运动后，再补一个加法修正**。

整个求值可以浓缩成一条链：

```text
提取本帧根运动 → 算目标偏差 → [预测缩放] → [弹簧补差] → 写回根运动属性
```

两个环节各管一摊：

1. **预测缩放（scaling）**。向前看 `AnimatedTargetTime` 秒，采样动画未来的根运动，得到它「本来还要转多少」——记作 `PredictedYaw`。如果动画确实在转（`|PredictedYaw| > 角度阈值`），就算一个缩放比：

   ```text
   Ratio = clamp( YawToTarget / PredictedYaw, MinScaleRatio, MaxScaleRatio )
   ```

   然后把这**一帧**根运动的旋转按 `Ratio` 缩放。动画转得不够就放大，转过头就缩小。这是「大头」：大部分轨迹修正靠它一步到位。

2. **弹簧补差（additive spring）**。缩放之后还剩下的偏差，交给一个临界阻尼弹簧逐帧追。弹簧的时间尺度来自 `ProceduralTargetTime`（转成半衰期），它输出一个增量旋转 `LinearCorrection`，**叠加**在缩放后的旋转上。这是「残差」：钳制漏掉的部分、以及缩放不生效时的全部误差，都由它慢慢消化。

最后把改好的旋转写回根运动属性流（`OverrideRootMotion`）。注意关键词是「写回属性流」——Steering 改的是「这一帧整体朝哪走」，不是任何一根骨头的姿势。

## 关键流程：每一步在算什么

### 目标偏差相对谁

第一步先定参考空间。节点在 Update 阶段拿一个 `RootBoneTransform`：如果上游有 Offset Root Bone（通过 `FRootOffsetProvider` 图消息），就用它给出的根骨变换；否则退回组件变换。目标偏差就定义在这个空间里：

```text
DeltaToTarget = RootBoneRotation⁻¹ · TargetOrientation
```

也就是「目标朝向相对当前根骨朝向」还差多少。正因为参考空间跟着 Offset Root Bone 走，Steering 的误差计算会和「网格暂时没跟上胶囊」的当前姿势保持一致，不会和 Offset Root Bone 打架。

### 预测需要资产 + 播放时间

缩放这一步有个前提：节点得知道「当前正在播哪段动画、播到哪一帧」，才能向前采样根运动。这靠 `CurrentAnimAsset` 和 `CurrentAnimAssetTime` 两个**瞬态引脚**，它们不是手动填的，而是由上游喂入——在 Motion Matching 的 Blend Stack 里，就是把当前选中姿势的资产与播放时间接进来。`bMirrored` + `MirrorDataTable` 则保证镜像播放的资产也能正确预测。

这两个引脚没接好，缩放就退化成「只剩弹簧补差」，转向会显得拖泥带水。

### 什么时候什么都不做

和 Orientation Warping 一样，Steering 在「没有根运动」时静默跳过：

- 属性流里根本没有根运动 → `ExtractRootMotion` 失败，直接 return。
- 有根运动但速度太低 → 用本帧位移长度除以 Δt 算速度，低于 `DisableSteeringBelowSpeed` 就整体关闭（缩放 + 弹簧都关）。更低一档的 `DisableAdditiveBelowSpeed` 只关弹簧、保留缩放。

起步、停止、待机这些帧没有方向可言，强行拐只会扭坏轨迹。

### 为什么缩放只在「真的在转」时启用

如果未来 `AnimatedTargetTime` 内动画几乎不转（`|PredictedYaw| < RootMotionThreshold`），缩放没有意义——对着一个接近 0 的预测值算比值，数值会爆炸。所以这时跳过缩放，全部交给弹簧。同时比值被钳在 `[MinScaleRatio, MaxScaleRatio]`，超出钳制范围的误差也留给弹簧线性补偿。这就是「缩放管大头、弹簧管残差」的明确分工。

### 弹簧追的是「缩放后的残差」，不是原始误差

算完缩放后，代码把预测旋转也乘上 `Ratio`，再重算目标偏差：

```text
DeltaToTarget = (Predicted · Ratio)⁻¹ · RootBoneRotation⁻¹ · Target
```

也就是说，弹簧追的是「缩放之后还差多少」，而不是原始全量误差。否则缩放和弹簧会对同一笔误差重复补偿，轨迹会过头。

### 最短路径与钳制

目标偏航和预测偏航可能差到 ±360° 附近，代码先做一次 wrap（选最短路径方向）再算比值，避免「该转 350° 却被当成 −10° 反着拐」。

## 它不碰什么、和谁配合

Steering 在链路里是「轨迹层」，放对了才有效：

- **和 Orientation Warping 是两层**。Orientation Warping 改「腿和脊柱的局部姿势」，让下肢朝移动方向；Steering 改「整体轨迹」，让根运动拐向目标。一个掰姿势、一个掰轨迹，可叠加但不重叠。边界见 [Orientation Warping](./Orientation-Warping.md)。
- **和 Motion Warping 分工**。精确到达交互点（翻越、处决、开门）用 Motion Warping，它有明确终点和截止时间；连续运动里跟随意图方向用 Steering，它没有硬性终点，是「边跑边修正」。见 [Motion Warping](./Motion-Warping.md)。
- **常驻 Motion Matching 的 Blend Stack 内层**。每个候选姿势在混合前先被 Steering 修正，保证混合出的姿势轨迹已经指向意图。放内层（而不是主 AnimGraph 再叠一层）避免对同一轨迹改两次；这也是 `CurrentAnimAsset` / `CurrentAnimAssetTime` 能拿到当前姿势资产与时间的原因。
- **参考空间跟随 Offset Root Bone**。有 Offset Root Bone 时，目标偏差相对「根骨当前姿势」来算；没有就退回组件空间。

## 常见问题

- **节点「没反应」**：先确认上游真的写了根运动属性——Steering 依赖根运动属性流，上游必须是 Motion Matching 或开了 Root Motion 的 Sequence Player。没根运动，`ExtractRootMotion` 失败就静默跳过。
- **转向慢半拍 / 有弹簧感**：多半是缩放没生效。检查 `CurrentAnimAsset` / `CurrentAnimAssetTime` 是否接好、`AnimatedTargetTime` 采样窗口内动画是否真的在转（低于 `RootMotionThreshold` 会被跳过）。只靠弹簧补差时，收敛速度受 `ProceduralTargetTime` 支配。
- **转向过头或来回摆**：弹簧时间尺度太小（`ProceduralTargetTime` 过小）或缩放比钳制范围太宽，缩放和弹簧叠加过量。确认弹簧追的是缩放后的残差、且比值落在钳制范围内。
- **起步 / 停止瞬间乱拐**：检查速度门槛。想整体关就调 `DisableSteeringBelowSpeed`；只想在低速时保留缩放、单独关弹簧，用 `DisableAdditiveBelowSpeed`。
- **用了 Offset Root Bone 后参考方向不对**：确认 Offset Root Bone 在 Steering 上游。Steering 会优先用它提供的根骨变换做参考，目标朝向的参考空间要与之匹配。

## 参考资料

- [Game Animation Sample Project](https://dev.epicgames.com/documentation/en-us/unreal-engine/game-animation-sample-project-in-unreal-engine)（Blend Stack Graph 内使用 Steering 的官方示例）
- [AnimNode_Steering — Python API](https://dev.epicgames.com/documentation/en-us/unreal-engine/python-api/class/AnimNode_Steering)（节点引脚速查）
- 引擎源码（UE 5.8）：`Engine/Plugins/Animation/AnimationWarping/Source/Runtime/`（`FAnimNode_Steering` 位于 `BoneControllers/AnimNode_Steering.h/.cpp`；根运动属性流接口见 `Engine/Source/Runtime/Engine/Public/Animation/AnimRootMotionProvider.h`；参考空间消息 `FRootOffsetProvider` 见 `BoneControllers/AnimNode_OffsetRootBone.h`）
