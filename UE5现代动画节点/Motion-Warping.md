# Motion Warping 详解

> Motion Warping 做的事一句话就能说清：在一段根运动动画里圈出一段「时间窗口」，把窗口内剩下的根运动平移与旋转重新分配，让角色在窗口结束时精确落在某个目标 Transform 上。它不像 Orientation Warping 那样掰姿势，而是直接改写「角色这一帧往哪走、转多少」。

## 它解决的问题

根运动动画是「一段固定的位移」。翻越障碍、跨上台阶、跳上掩体时，动画里预先烤好的落点几乎永远对不上实际情况：障碍物高一点、敌人站位偏一点，角色要么扑空、要么脚陷进地里。要精准落位，传统做法是给每段动画做大量变体或 Blend Space，资产爆炸且难以维护。

Motion Warping 的答案是：**不换动画，只改「这段动画在这段时间里产生的位移」**。它把「动画本来会走到的位置」和「我实际想让你到的位置」之间的差距，分摊到窗口剩余的时间里逐帧消化，最终在动画的某个确定时刻精确对齐目标点。

它明确**不做**三件事，理解了边界才不会误用：

- **不改姿势**：它不碰骨骼，不调整手脚，只改根运动的平移和旋转。脚踩歪了、手没够到，那是 IK 或资产本身的事。
- **不持续追踪**：它服务的是「离散交互点」——一个明确的落点、一个要对齐的物体，动画走到窗口末尾就结束。连续的跟随意图是 Steering 的活。
- **不驱动胶囊以外的运动**：它改写的是根运动，所以前提是角色真的在靠根运动移动（见后文「为什么必须开 Root Motion」）。

## 核心思想：重分配一段窗口内的根运动

整个机制可以浓缩成一句话：**把「窗口结束时应该在哪」当成约束，逐帧改写「这一帧该迈多大、转多少」，使累积结果在窗口末恰好满足约束。**

设窗口从 `StartTime` 到 `EndTime`。每一帧，Modifier 会算两样东西：

- 窗口内**剩余**的根运动 `RootMotionTotal`：从当前时间到 `EndTime` 之间，动画本来还会走多远、转多少。
- 这一帧的份额 `RootMotionDelta`：当前帧到下一帧之间的那一段。

然后它把「当前位置到目标点」的这段差距，代替「本来会走到的位置」，重新映射回这一帧的位移。目标点越远、窗口越短，单帧修正量就越大——这正是「允许多大修正」的根源，下一节展开。

旋转同理：不直接跳到目标朝向，而是让角色的旋转在窗口结束前追平目标旋转（默认按 Slerp 插值，也可选恒定角速度或整体缩放）。

## 三件套怎么协作

Motion Warping **不是单个 AnimNode**，而是「组件 + Montage NotifyState + Root Motion Modifier」三件套。理解三者分工是读懂这套系统的关键：

| 角色 | 是什么 | 负责什么 |
|---|---|---|
| `UMotionWarpingComponent` | 挂在角色上的组件 | 保存同名 Warp Target 表、持有活跃的 Modifier 列表、接管根运动改写入口 |
| `UAnimNotifyState_MotionWarping` | 摆在动画上的 NotifyState | 用自身起止时间定义 Warping Window，并携带一个 Modifier 模板 |
| `URootMotionModifier`（及其子类） | 被 Notify 持有的配置对象 | 窗口内逐帧重写根运动平移/旋转 |

它们靠一个**名字（`WarpTargetName`）**串起来：游戏侧往组件里写一个「目标点」（名字 + Transform），动画侧的 Notify 在它的 Modifier 上写明「我要对齐哪个名字的目标」。窗口激活时，Modifier 按名字去组件里查表。

### 完整流程（每帧）

1. **入口在 CharacterMovementComponent**。角色移动组件在把本地根运动转成世界位移之前，会调用一个回调 `ProcessRootMotionPreConvertToWorld`。Motion Warping 的 Adapter 挂在这个回调上，把「当前在放哪段 montage、播到第几帧、权重、播放速率」打包成上下文，转交给组件。
2. **组件扫描窗口**。组件遍历当前动画的 Notify，发现 `UAnimNotifyState_MotionWarping`，且当前时间刚进入 `[StartTime, EndTime)`，就调用 `OnBecomeRelevant`。
3. **Notify 复制一个 Modifier 实例**。Notify 身上的 `RootMotionModifier` 是一个**模板**（默认是 Skew Warp）。组件把它复制成一份独立实例，写入动画名和窗口起止时间，加进自己的 Modifier 列表，同时把 Notify 的 `OnWarpBegin / OnWarpUpdate / OnWarpEnd` 绑定到实例的生命周期回调上。
4. **Modifier 进入状态机**。Modifier 有四种状态：`Waiting → Active → MarkedForRemoval`（或 `Disabled`）。进入窗口前等待；进入后激活；窗口过掉、动画被切走、或播放位置被手动跳走时标记移除。
5. **激活瞬间记快照**。第一次激活时，Modifier 记下角色当前的位置/朝向（`StartTransform`）、实际激活时间，以及整个窗口内的总根运动 `TotalRootMotionWithinWindow`。
6. **逐帧重写**。每个 Active 的 Modifier 依次对根运动调 `ProcessRootMotion`，改写平移与旋转；所有 Modifier 处理完，结果才交还给移动组件转成世界位移。

关键点在于：**Warp Target 存在组件里，而不是动画里**。同一段动画可以被复用到不同落点上——游戏运行时先把目标点写进组件，再播 montage，Modifier 按名字取到最新目标。目标可以是一帧前刚算出来的交互点，也可以是跟随某个移动组件实时刷新的点（`bFollowComponent`）。

## Warping Window 的时间边界，为什么决定「允许多大修正」

Warping Window 不是随意画的框，它是**修正量的预算表**。Modifier 的约束只有一条：窗口结束时到目标点。中间的每帧位移是「剩余差距 ÷ 剩余时间」摊出来的，所以：

- **窗口越短，单帧修正越狠**。同样的落点差距，被挤进更少的帧里，每帧的位移方向被掰得越陡。窗口只有两三帧时，角色几乎是被「瞬移式甩过去」的，这就是突兀感的来源。
- **目标差距越大，偏折角越大**。Skew Warp 求的偏斜角，本质是「动画本来会走到的方向」和「实际要到目标点的方向」之间的夹角（源码里就是两个方向向量点积反余弦）。差距越大，这个角越大，轨迹弯得越明显。
- **窗口末尾是硬约束**。旋转侧最直观：默认 Slerp 模式的插值参数是「本帧经过的（带播放速率）时长 ÷ 剩余窗口时长」，所以旋转**恰好在窗口结束时追平目标朝向**——窗口短就转得急。

两个参数直接印证这套逻辑：

- `WarpRotationTimeMultiplier`：源码注释写得明白——窗口 2 秒、该值 0.5，目标朝向就在 1 秒内达成而非 2 秒。等于「把预算提前花完」。
- `MaxSpeedClampRatio`：把 warp 后的速度钳制在动画原始速度的某个倍数内，防止目标太远导致单帧位移暴走。

所以调窗口时的经验法则：**给修正留够时间，目标点别离动画自然落点太远**。动画的根运动方向本来就和目标方向接近时，偏折角小、修正几乎不可见；反之，无论用什么算法，都救不回「位移不够硬凑」的别扭感。

## Skew Warp 的核心思想：不是「拉直线」，而是「掰弯」

Motion Warping 早期的 Simple Warp 已经被标记 Deprecated，原因是它做修正太粗暴：它把这一帧的水平位移方向**直接替换成「指向目标点的方向」**，长度按比例缩放。结果横向的摇摆、弧线这些动画原本的运动质感全被抹掉，只剩一条直冲目标的直线。

Skew Warp 的思路不同：**保留动画本来的横向/上下分量，只把「朝前」的分量掰向目标**。它先建立一个以「剩余根运动方向」为前向的坐标系 `ToRootSyncSpace`，然后构造一个「缩放 × 剪切」矩阵：

```text
Skewed = ScaleMatrix · ShearXAlongYMatrix · ShearXAlongZMatrix
```

- `ScaleMatrix` 只缩放 X（前向）分量：把前向位移拉长/缩短到足以够到目标；
- 两个剪切矩阵把 X 分量「推」向 Y（左右）和 Z（上下），推的量由「到目标方向」相对「到原落点方向」的偏角决定。

剪切矩阵只改 X、不动 Y 和 Z，所以**垂直于前进方向的摆动被原样保留**——角色边迈步边左右摆的节奏还在，只是整条轨迹被「掰」得弯向目标。这就是它叫 Skew（倾斜/剪切）的原因：不是替换方向，而是给运动加一个渐变的偏折。

当动画本身**没有**平移根运动时（比如原地跳），走另一条路：从 `StartTransform` 到目标点按缓动曲线直接插值补平移，配合可选的缓动函数（`AddTranslationEasingFunc`）。

## 为什么必须开 Root Motion

Motion Warping 的入口是 `CharacterMovementComponent::ProcessRootMotionPreConvertToWorld`，它只在角色**以根运动驱动位移**时被调用。整个改写发生在「本地根运动 → 世界位移」之间：

- 若动画没启用根运动，移动组件根本不走这条路径，Modifier 收不到上下文，也就无从改写；
- 即使强行拿到，没有根运动可改，角色的位移由速度驱动，改了也是白改。

编辑器里也能看到这套约束：Skew Warp 的 `DrawInEditor` 第一行就 early-out——动画没有根运动，直接不画。所以排查「没反应」时，先确认：动画资产开了 Force Root Lock / Root Motion、montage 启用了根运动、角色是 Character 且移动组件在消费根运动。

## 和 Steering、Offset Root Bone 的边界

三者常被混为一谈，但其实分工清晰：

- **Motion Warping = 精确到达**。「在动画的第 X 帧，角色的脚（或某个指定骨骼）必须落在这个 Transform 上」。它有明确的终点和明确的时间点，做完就结束，靠根运动达成。
- **Steering = 连续跟随意图**。「朝这个方向/这个目标持续移动」，没有固定的到达时刻，由移动组件的速度与朝向逻辑（或 Motion Matching 的轨迹）持续修正。它回答「往哪走」，不回答「第几帧到哪」。
- **Offset Root Bone = 视觉根与胶囊短暂分离**。「让网格根骨暂时偏离胶囊去够某个点，但胶囊的移动逻辑不变」。它是姿势层的偏移，胶囊该走哪还走哪；而 Motion Warping 改的是胶囊实际要去哪。

一句话边界：**要「在确定时刻钉到确定点」，用 Motion Warping；要「持续朝着意图走」，用 Steering；要「视觉上够一下但身体不真的挪过去」，用 Offset Root Bone。** 值得注意的是，Skew Warp 还支持把对齐点从「根」换成动画里的某根骨骼（`WarpPointAnimProvider = Bone`），这时对齐的是「手/脚在窗口末的位置」，根落在哪则由骨骼偏移反推——适合精确对齐手部命中点这类需求。

## 常见问题

- **完全没反应**：先查 Root Motion 链路——动画是否开根运动、montage 是否启用、角色是否 Character + 移动组件。再查名字：Modifier 上的 `WarpTargetName` 和运行时 `AddOrUpdateWarpTarget` 写入的名字必须完全一致（大小写敏感）。最后确认窗口时间：当前时间要真正落进 `[StartTime, EndTime)`。
- **对齐了但方向不对**：旋转对齐默认是「朝向 = 目标点的朝向」（`RotationType = Default`），若你只给了位置没给朝向，角色会转到目标 Transform 的默认朝向。想让角色「面向」目标点而非「朝向与目标一致」，用 `Facing` 模式。
- **落点差一点/过了**：检查 `bIgnoreZAxis`（默认忽略 Z，垂直位移不修正）和 `bWarpToFeetLocation`（对齐的是脚底还是 Actor 原点）。想要「动画结束时」而非「窗口结束时」到位，勾上 `bSubtractRemainingRootMotion`。
- **目标在动，跟丢一拍**：`bFollowComponent` 让目标逐帧刷新，但源码注释明确提示：若你的角色先于目标 Actor tick，会有一帧延迟——给目标 Actor 加 tick 前置依赖可消除。
- **修正突兀、瞬移感**：窗口太短或目标差距太大。拉长窗口、让目标靠近动画自然落点、用 `MaxSpeedClampRatio` 限制速度、调 `WarpRotationTimeMultiplier` 分摊旋转。
- **跳过动画片段导致窗口错乱**：Modifier 检测到播放位置被手动跳离窗口会自我标记移除（源码里有「CurrentPosition 被手动改变」的判定），这是有意为之——位置被人为跳走，再硬掰没有意义。

## 参考资料

- [Motion Warping](https://dev.epicgames.com/documentation/en-us/unreal-engine/motion-warping-in-unreal-engine)（官方文档）
- [Game Animation Sample Project](https://dev.epicgames.com/documentation/en-us/unreal-engine/game-animation-sample-project-in-unreal-engine)
- 引擎源码（UE 5.8）：`Engine/Plugins/Animation/MotionWarping/Source/MotionWarping/`——组件 `Public/MotionWarpingComponent.h`，NotifyState `Public/AnimNotifyState_MotionWarping.h`，Modifier 基类 `Public/RootMotionModifier.h`，Skew Warp `Public/RootMotionModifier_SkewWarp.h`，Character 接入点 `Public/MotionWarpingCharacterAdapter.h`
