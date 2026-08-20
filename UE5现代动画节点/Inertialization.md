# Inertialization 详解

> Inertialization 做的事一句话就能说清：在切换瞬间采样一次「旧姿态相对新姿态的差，以及这个差正在以多快的速度变化」，之后每帧只求值新姿态，把这笔「运动残差」逐帧衰减到零。它的聪明之处在于，旧动画从此不需要再被求值——它的影响被冻结成一组「位置/旋转 + 速度」，自己慢慢地散掉。

## 它解决的问题

普通的状态机过渡是「Standard Blend」：整个混合期内，源状态和目标状态**同时求值**，按权重加起来。这带来两个代价：

1. **成本翻倍**：过渡期间两套状态都在跑完整的 Evaluate，骨骼、曲线、Notify 全是双份。
2. **平均姿态**：两个姿态取平均，脚会打滑、手会穿模、运动会显得「发软」——尤其当两个姿态差得远、或者过渡时间很短时。

Inertialization 换了一条思路：**不在混合期持续求值来源姿态**。它只在切换的那一帧，把「上一帧的输出姿态」和「这一帧的新姿态」之间的差采样下来，顺带算出这个差的变化速度；之后把新姿态当底子，把采到的差当作一笔残差，用一条曲线把它从 100% 衰减到 0%。衰减期间，旧动画已经完全退场，什么都不用算。

所以它的本质是**外推**：用「旧动作这一瞬间还在往哪动」去平滑地覆盖「切到新动作」的跳变，而不是真的把两段动画揉在一起。

## 核心思想：采一次「差 + 速度」，然后只衰减残差

节点每帧都会把**自己的输出姿态**存一份快照（位置、旋转、缩放，外加曲线和根运动增量），只存当前帧和上一帧两份。当收到请求时，它做三件事：

1. **算差**：对每根骨骼，算出 `旧输出 − 新姿态` 的差。平移得到方向+大小，旋转得到轴+角度，缩放同理。
2. **算速度**：用「上一帧的差」和「再上一帧的差」的差分，推出这个差正在以多快的速度变化。也就是说，残差不只是「静止地淡出」，而是**带着旧动作的动量**继续滑一段。
3. **衰减**：把残差按一条曲线逐帧缩小，加到新姿态上；曲线走完，输出就纯粹是新姿态。

写成一条式子的味道（以单根骨骼的平移为例）：

```text
输出 = 新姿态 + 方向 × F( 差值大小, 差值速度, 已过时间, 时长 )
```

`F` 在时间 0 处返回 1（也就是完全等于旧姿态，切换瞬间零跳变），在时长终点处连位置、速度、加速度一起归零。这样残差消失时不会有新的顿挫。

**为什么旧动画能就此停掉？** 因为这套机制从头到尾只需要「切换瞬间」这一个采样点。旧动作后续会怎么发展，是用「速度」这一项外推出来的，不需要真的去播旧动画。于是状态机可以把旧状态立刻标记为不活跃，过渡期只剩一个新状态在求值——这正是省成本的地方（状态机过渡类型的提示里就写着：Inertialization 下「同一时刻只有一个状态活跃」）。

## 关键流程：请求 → 采样 → 衰减

节点内部是一个三态小状态机：

- **Inactive**：没收到请求，纯透传，顺带每帧刷新快照。
- **Pending**：收到了请求，下一次 Evaluate 时采样「差 + 速度」，随即转入 Active。
- **Active**：每帧把残差按曲线衰减后加到新姿态上，直到时长走完，回到 Inactive。

衰减用的曲线是一条**五次多项式**，专门为这个场景调过：

- 起点满足「值 = 差、一阶导 = 速度」，终点满足「值、一阶导、二阶导全为 0」，所以收尾干净不顿。
- 如果初速度是「背离零」的方向（会让残差先变大再回落，出现来回甩），直接把速度钳到 0。
- 如果速度太大、按原时长走会「冲过头」穿过零再弹回来，就把实际时长**自动缩短**，保证残差单调归零、绝不越界。

这条曲线的细节解释了最常见的两个调参手感：**时长太短 → 残差被快速掐断，表现为 Pop；时长太长 → 旧动作的动量拖得久，表现为「旧动作不肯走」**。

两个值得知道的工程细节：

- **逐骨骼时长**：请求可以带 Blend Profile，让不同关节用不同的衰减时长（比如脚踝快速跟上、脊柱慢一点）。没有 Profile 时，所有骨骼用同一个时长。
- **Deficit（时间赤字）与「姿态融化」**：如果一次 Active 还没走完就被新的请求打断，剩余时间就「作废」了。反复这样打断，新姿态还没稳定就又被外推，姿态会越来越飘——源码里叫 pose melting。节点会记录被打断时剩了多少时间（deficit），下次请求自动把这个缺口从时长里扣掉，来抑制频繁打断造成的融化。

## 它靠 Request 触发，没请求就不动

这是最容易误解的一点：**Inertialization 节点自己不会「自动」做任何事**。它只是一个等待请求的接收端，请求不来，它永远 Inactive，纯粹透传。

请求从哪来？两处最典型：

- **状态机过渡**：把某条 Transition 的 Blend Logic 设为 **Inertialization**。切换触发时，状态机节点发出一条请求，把过渡时长（Crossfade Duration）、Blend Profile、Blend Mode 等一并带上。顺带一提，标准混合也有「退回 Inertialization」的开关：当重入一个仍在活跃的状态、而该状态会重置时，为避免 Pop 自动改用惯性化。
- **Linked Anim Graph / Linked Anim Layer**：图与图之间切换时的 Blend In / Blend Out 时长，也是通过同一种请求机制发出来的（必须在其后接 Inertialization 节点才会生效）。

请求的传输方式是一条**图消息**：Inertialization 节点在 Update 时把自己注册成一个「请求接收器」，包裹住它的 Source 子树做 Update；子树里任何想惯性化的节点（状态机过渡、Linked Anim Graph……）往上一层 `GetMessage` 就能找到它，把请求投递进去。这意味着：

- **节点必须放在请求能到达的输出路径上**——即请求发生处的「祖先」方向（rootwards）。放错了，请求找不到接收器，编辑器会报错「No Inertialization node found…… Add an Inertialization node after this request」。
- 同一帧多个请求会合并，**每个骨骼取最短的请求时长**；多个请求也可能被同一个节点接收。
- 图里有多个 Inertialization 节点时，可以用 Tag 把请求路由到指定节点（节点的 Tag 与请求的 Tag 不匹配时，请求会被丢弃）。

## 和谁配合、还有几个细节

- **和 Dead Blending 同源**：Dead Blending 节点实现的是同一套 `IInertializationRequester` 接口和同一套请求/衰减管线，差别在残差的表达方式。两者都能接收同一种请求，选其一即可。
- **根运动也一起惯性化**：节点不只是对骨骼做残差，还会把「根运动速度差」一起采样并衰减，保证惯性化期间位移/朝向的跳变同样被抹平。因此它对带 Root Motion 的 Sequence Player 也适用。
- **自动检测传送**：如果根骨骼的世界位置一帧内跳变超过组件传送阈值，节点会取消进行中的请求、并清空速度历史——传送不该被当成「一次大速度」来外推。
- **世界空间 / 世界旋转模式**：默认在局部空间惯性化；当角色换了挂接父级（上/下载具这类瞬间换参考系）或组件朝向发生突变时，节点会自动切换到世界空间或世界旋转模式去抹平这种不连续。
- **过滤器**：可以指定某些骨骼或曲线不参与惯性化——它们会在切换瞬间立刻跳变到新值（适合那些「宁可硬切也不要拖泥带水」的元素）。
- **Reset on Becoming Relevant**：节点若刚变成相关（比如从被缓存跳过到重新参与求值），可选地清掉请求和状态，避免把陈旧的混合带进来。

## 常见问题

- **「没生效 / 没反应」**：先确认真的有请求——过渡的 Blend Logic 是不是 Inertialization（或 Linked Anim Graph 的 Blend 时长是否已配）。节点本身不产生行为，没请求就永远 Inactive。
- **报「No Inertialization node found」**：节点没放在请求的祖先路径上。把它挪到状态机 / Linked Anim Graph 的外侧（更靠近输出根的方向）。
- **切换瞬间 Pop**：大概率时长太短，残差还没衰减就被掐断；或者该骨骼/曲线被设成了过滤器，直接硬切了。调大时长、检查过滤器。
- **切过去后「旧动作拖着不走」**：时长太长，旧动作的动量外推得过久。缩短过渡时长，或针对性地给需要快速跟上的骨骼配 Blend Profile。
- **快速连打、姿态越来越飘**：这是反复打断造成的 pose melting。源码用 deficit 机制做了抑制，但极端频繁的打断仍可能表现出来；适当放宽切换条件、避免同一帧内反复触发请求。
- **传送后出现诡异的大位移**：确认传送阈值配置正常，节点会据此取消请求并重置速度；若阈值过小/过大，自动检测会失准。

## 参考资料

- [Inertialization node - Unreal Engine Documentation](https://dev.epicgames.com/documentation/en-us/unreal-engine/inertialization-node-in-unreal-engine)
- [State Machine transitions (Blend Logic) - Unreal Engine Documentation](https://dev.epicgames.com/documentation/en-us/unreal-engine/state-machines-in-unreal-engine)
- 引擎源码（UE 5.8）：
  - `Engine/Source/Runtime/Engine/Classes/Animation/AnimNode_Inertialization.h`（节点声明、状态机与快照结构）
  - `Engine/Source/Runtime/Engine/Private/Animation/AnimNode_Inertialization.cpp`（`CalcInertialFloat` 五次多项式、`InitFrom`/`ApplyTo`、deficit 与传送检测）
  - `Engine/Source/Runtime/Engine/Classes/Animation/AnimInertializationRequest.h`（请求结构）
  - 请求发起方：`Engine/Source/Runtime/Engine/Private/Animation/AnimNode_StateMachine.cpp`（过渡 `TLT_Inertialization`）、`AnimNode_LinkedAnimGraph.cpp`
