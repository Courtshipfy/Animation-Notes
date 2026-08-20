# Trajectory 详解

> Trajectory 做的事一句话就能说清：把「角色刚才怎么走的」和「接下来打算怎么走」编码成一串带时间戳的位置/朝向采样点，交给 Motion Matching 去和候选动画自身的根运动轨迹逐点比。它是查询意图的「数据 + 预测」，不是输出姿势的节点——它描述你想去哪，最终摆什么姿势是 PoseSearch 的事。

## 它解决的问题

Motion Matching 的核心动作是「根据当前状态，从数据库里挑一段最合适的动画」。问题是：只看当前这一帧的姿态和速度，信息不够。同样是跑步的姿势，下一秒可能是继续直跑、可能是急转、可能是突然刹停——而候选动画必须提前知道你「要往哪去」，才可能在身体真正转向前就把那段转身动画铺进去。

Trajectory 就是用来补这个「未来意图」的。它把角色的运动意图摊成一条时间轴：过去的几个点是「我从哪来」，当前点是「我现在在哪」，未来的几个点是「我接下来想往哪去」。匹配时，数据库里每一段动画也有它自己的根运动轨迹（脚踩出来的实际位移），谁的轨迹和这条意图轨迹最像，谁就最符合玩家当下的操作。

它明确**不做**的事，理解边界才不会误用：

- **不输出姿势**：它不产生任何骨骼变换，也不出现在 AnimGraph 的输出链上。它是一份数据，由 AnimBP 里的函数调用（`Pose Search Generate Trajectory`）生成并缓存下来，供后面的 PoseSearch 节点读取。
- **不驱动 Actor**：预测出的未来点只是「假设移动组件继续这么走」的推演，胶囊真正的移动仍由 CharacterMovementComponent / Mover 决定，Trajectory 只是照着它的当前状态做外推。
- **不决定选谁**：它只负责把「意图」编码进查询向量，最终谁胜出是 Trajectory Channel 里那一堆 Position/Velocity/Heading 子通道加权比出来的。

## 核心思路：一串「过去 + 预测未来」的采样点

Trajectory 的数据结构非常朴素，就是一个有序数组，每个元素是：

```text
采样点 = { 位置 Position, 朝向 Facing, 时间 TimeInSeconds }
```

位置是世界空间的 `FVector`，朝向是世界空间的 `FQuat`，时间决定它属于哪一段：

- **时间 < 0**：历史采样点，记录角色真实走过的位置；
- **时间 = 0**：当前点；
- **时间 > 0**：预测采样点，从当前状态向前推演。

「过去」和「未来」被放进**同一个数组、同一个世界坐标系**里保存，这是它最关键的约定——正因为两者共享坐标系，查询时才可以把「我实际走过的路线」和「我预测要走的路线」拼成一条连续的意图轨迹，再去和候选动画的根运动轨迹做逐点比较。

另一层要理解的是「过去」和「未来」的来源完全不同：

- **历史是「记」出来的**：每隔一个采样间隔，把角色网格当前的世界位置/朝向抄进历史环里，是真实发生的位移；
- **预测是「算」出来的**：把当前速度、加速度、朝向代入一个地面移动模型，一步一拍向前积分，是还没有发生的位移。

这也解释了为什么它叫「数据 + 预测」：一半是事实，一半是假设，共同构成一份完整的查询输入。

> UE 5.6 起，旧类型 `FPoseSearchQueryTrajectory`（字段 `AccumulatedSeconds`）已废弃，统一迁移到引擎级的 `FTransformTrajectory`（字段 `TimeInSeconds`），定义在 `Engine/Source/Runtime/Engine/Public/Animation/TrajectoryTypes.h`。写蓝图或 C++ 时认准新类型即可。

## 关键流程

### 第一步：采集当前状态

每帧生成轨迹前，先从角色身上抓一份「快照」：网格组件的位置和朝向、移动组件的速度和当前加速度、以及控制器这一帧的 yaw 变化率。朝向有个细节——默认取网格组件的旋转，若角色不随移动旋转（`bOrientRotationToMovement` 关闭）则改用控制器 yaw 加网格相对旋转，保证轨迹朝向和玩家「看到的方向」一致。

### 第二步：滚历史（过去）

历史用一个固定长度的环形窗口维护。到了该记录的时机，就把上一帧的位置/朝向推进去，把最老的那个挤出去；没到时机就只更新各点的时间戳。这里藏着一个容易被忽略的处理：**地面（移动平台）修正**。角色站在电梯/旋转平台上时，网格的世界位移里混进了平台的位移，而匹配关心的是角色「自己」的移动意图。所以实现会把「平台带来的平移/旋转」从历史里剥掉——平移平台用普通版本即可，旋转平台必须用带旋转的版本，否则历史朝向会越积越偏。

### 第三步：推未来（预测）

预测是对当前状态的纯前向积分，循环里每一步做的事可以概括成：

```text
位置  += 速度 × 步长
速度   = 地面移动模型一步积分( 速度, 加速度 )     // 复用 CMC 的摩擦/制动/加速逻辑
朝向   = 控制器 yaw 变化率 × 步长 旋转朝向          // 再可选地朝加速度方向对齐
```

关键在于：速度更新**复用 CharacterMovementComponent 的移动数学**（制动减速度、地面摩擦、最大速度钳制）。这让预测轨迹和胶囊「真的会怎么走」高度一致——你会刹停在预测里也会刹停，你会被最大速度封顶在预测里也封顶。此外还有几个「整形」旋钮：把速度方向朝加速度方向掰一点（`BendVelocityTowardsAcceleration`）让转弯更利落、用曲线重映射速度/加速度大小（动画表现速度和实际速度不一致时用）、限制控制器 yaw 变化率避免镜头甩太快。

### 第四步：交给 Trajectory Channel 逐点比

Trajectory 生成完，本身不干任何事，真正的消费端是 **Trajectory Channel**。它是一个「组合通道」：你在它里面配置若干个时间偏移（Offset），每个偏移会展开成一个或多个子通道，去匹配「这个时间偏移处」的信息：

- `Position`：该时刻的位置；
- `Velocity`：该时刻的速度（可只比方向）；
- `FacingDirection`：该时刻的朝向。

查询时，每个子通道在 Offset 时刻取「角色根骨的世界变换」——也就是根骨局部姿态（根骨处几乎恒等）乘上该时刻轨迹给定的世界变换——编码进查询向量；数据库索引时，用完全相同的方式取「候选动画在该时间偏移处的根运动变换」。两者逐点比误差、加权求和，误差小者胜出。

默认配置就同时挂了「过去一点 + 当前 + 未来两点」，所以它天然在比较一整条意图轨迹：过去的点保证连续性（别让新选中的动画和来路对不上），未来的点保证响应性（提前选出转向/起步动画）。

## 和谁配合

Trajectory 是链路里承上启下的一环，位置对了才有效：

- **上游靠移动组件**：预测的输入（速度/加速度/摩擦/最大速度）全部来自 CharacterMovementComponent 或等价的 Mover，所以移动参数调得对不对，直接决定轨迹准不准。
- **下游进 PoseSearch / Motion Matching**：轨迹被塞进 `IPoseHistory`（姿势历史），随搜索上下文一起交给数据库。姿势历史里同时存着「过去几帧的混合姿势」和「轨迹给的世界变换」，Trajectory Channel 正是通过它取到各时间偏移的根变换。
- **可选接 Motion Warping**：有 montage 在播时，可以改用 montage 的根运动（含 warp 修正后的位移）来做预测，让被 warp 的路径也反映在意图轨迹里。
- **非 Character 可走 Predictor 接口**：Mover 或自研移动不想走 CMC 时，实现 `IPoseSearchTrajectoryPredictorInterface` 自己提供位置/朝向/速度/角速度，由它来 `Predict`。
- **可选做世界碰撞/重力修正**：`Handle Trajectory World Collisions` 能把重力下落、落地时间、贴地高度折进预测点，用于跳跃/落地场景。

## 常见坑

- **每帧突然改向，数据库再全也频繁重选（最该先想清楚的一条）**。预测是对「当前瞬时状态」的纯前向积分，输入本身**没有做平滑**——只有控制器 yaw 变化率钳制、`BendVelocityTowardsAcceleration`、`RotateTowardsMovementSpeed` 这几处局部整形。手柄猛地从左推到右时，未来几个点会瞬间甩到新方向，查询向量跳变，匹配就会不停地换动画，表现成「滑步/抽帧」。这时候**先做移动输入的平滑与去抖、调移动模型，再考虑堆数据库动画数量**，顺序反了收效甚微。
- **朝向坐标空间配错**。`Facing` 到底取网格旋转还是控制器 yaw，取决于是否随移动旋转。配错了，轨迹朝向和身体实际面向脱节，选出的转向动画总是「慢半拍」或「转向量不对」。
- **旋转平台上一停就漂**。平移平台用 `UpdateHistory_TransformHistory` 已经能把平台位移剔除，但旋转平台必须用带旋转的 `WithRotation` 版本，否则历史朝向会跟着平台转、越积越偏。
- **乱用速度/加速度重映射曲线**。`SpeedRemappingCurve` / `AccelerationRemappingCurve` 是给「动画视觉速度 ≠ 实际移动速度」的特殊情况用的，日常 locomotion 乱挂会直接破坏预测与 CMC 的一致性，轨迹反而失真。
- **预测步长与跨度不对**。预测间隔太大，未来点稀疏、意图糊成一片；整体时间跨度太短，选不出「该提前转身」的动画。这两个参数要和数据库动画的节奏一起调。
- **类型混淆**。5.6 之后还在用 `FPoseSearchQueryTrajectory` 的旧字段名（`AccumulatedSeconds`），会遇到废弃告警；统一切到 `FTransformTrajectory` / `TimeInSeconds`。

## 常见问题

- **轨迹会直接改变角色姿势吗？** 不会。它只生成数据，最终姿势由 PoseSearch 根据它选出的动画决定。想验证，看 AnimGraph：Trajectory 只有函数调用，没有连接输出姿势的引脚。
- **加了 Trajectory 还是选不出转向动画？** 先查三点：预测点数和时间跨度够不够、朝向坐标空间对不对、输入是不是在跳变（最常见）。数据库规模往往是最后才考虑的因素。
- **站在移动平台上一停，轨迹就往前漂？** 历史没做地面/旋转修正，旋转平台务必用带旋转的更新版本。
- **预测点看起来乱、抖动？** 检查是否误挂了速度/加速度重映射曲线，以及 `BendVelocityTowardsAcceleration` 是否设得过大。

## 参考资料

- [Motion Matching in Unreal Engine](https://dev.epicgames.com/documentation/en-us/unreal-engine/motion-matching-in-unreal-engine)
- [Pose Search in Unreal Engine](https://dev.epicgames.com/documentation/en-us/unreal-engine/pose-search-in-unreal-engine)
- [Game Animation Sample Project](https://dev.epicgames.com/documentation/en-us/unreal-engine/game-animation-sample-project-in-unreal-engine)
- 引擎源码（UE 5.8）：轨迹数据结构 `Engine/Source/Runtime/Engine/Public/Animation/TrajectoryTypes.h`；生成与预测 `Engine/Plugins/Animation/PoseSearch/Source/Runtime/Public/PoseSearch/PoseSearchTrajectoryLibrary.h` 与 `Private/PoseSearchTrajectoryLibrary.cpp`（预测移动模型见 `StepCharacterMovementGroundPrediction`）；预测器接口 `Public/PoseSearch/PoseSearchTrajectoryPredictor.h`；消费端 `Public/PoseSearch/PoseSearchFeatureChannel_Trajectory.h` 与 `Private/PoseSearchFeatureChannel_Trajectory.cpp`（组合通道展开成 Position/Velocity/Heading 子通道）。
