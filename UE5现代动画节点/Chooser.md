# Chooser 详解

> Chooser 不是动画节点，也不是姿势工具，而是一个「规则表驱动的对象选择框架」：它从上下文对象里读字段值，逐列筛掉不匹配的行，最后把命中行返回的对象交出去。你可以把它当成把散落各处的 if/switch 收拢成一张可编辑的数据表。

## 它解决的问题

在动画蓝图里，我们经常要根据一堆运行时状态选东西：是站是趴、拿没拿武器、移动速度多少、处于哪个 GameplayTag 状态……传统写法是一串 State Machine、Blend Space 加一堆 bool 分支。状态一多，判断逻辑就散落在几十个节点和蓝图里：改一条规则要到处翻，新加一个「移动模式」维度就要重写一整片判断。

Chooser 把这个「根据输入选输出」的决策抽成一个独立资产：一张规则表。输入来自上下文对象（AnimInstance、Pawn，或任意结构体）上的字段，每一行是一个候选结果，每一列是一条过滤/输出规则。改规则就是改表格，加维度就是加一列。

明确它**不做**什么，才能用对：

- **不算姿势、不做 IK**：它只负责「选哪个资产」，返回的是一段动画、一个 PoseSearch Database、一个蒙太奇或一个类，绝不改姿势、不解算骨骼。
- **不选「姿势」本身**：它选的是「资源」，不是某一帧姿势。真正的姿势挑选交给 PoseSearch / Motion Matching。
- **不混合、不播放**：它只是缩小/决定「允许哪些资源」，拿到资产之后的混合与播放是 `FAnimNode_ChooserPlayer`（一个基于 BlendStack 的播放器）的活。

## 核心模型：一张规则表

`UChooserTable` 资产由三块组成：

1. **上下文声明（Parameters / ContextData）**：这张表允许从哪些类/结构体上读字段（比如 AnimInstance、Character、一个移动状态 struct）。它是「输入有哪些」的白名单，也是列里属性绑定的合法起点。
2. **列（Columns）**：每列绑定上下文里的一个字段（通过一条属性链，形如 `CharacterMovement → MovementMode`），并对每一行存一个判断值。列分三类——**过滤列**读值并剔除不匹配的行，**评分列**给每行累加一个代价用于排序，**输出列**把行里存的值写回上下文。判断列在编辑器里就是「一列」，候选结果就是「行」，编辑时呈现为一张电子表格。
3. **结果行（Results）+ 兜底（FallbackResult）**：每行是一个结果——硬引用资产、软引用、类、或嵌套 Chooser。所有行都不命中时用 FallbackResult；没配 Fallback 就返回 null。

关键点：列是「读字段 → 比较 → 收窄行集」的谓词集合，结果行是「命中后返回什么」。字段名、比较值、结果引用全都是数据，不是代码——这是它和手写 if/switch 的本质区别。

另外注意结果分三类（`EObjectChooserResultType`）：返回「某个类的对象」、返回「某个类的子类」（比如要一个可生成的 Character 类型）、或干脆不返回主对象、只写输出值。嵌套 Chooser（`FEvaluateChooser` / `FNestedChooser`）让一行命中后可以递归评估另一张表，同一份上下文继续往下传，靠 `RootChooser` 让嵌套表共享根表的上下文定义。

## 关键流程：从上下文到结果

运行时评估（`EvaluateChooser`）是一条一遍过的过滤流水线：

1. 把当前上下文（`FChooserEvaluationContext`，里面装着 AnimInstance 对象 + 可选结构体）对应到所有未禁用的行，收集成一个索引列表，每项是 `{行号, 代价}`。
2. 按列顺序逐列过滤：过滤列读自己绑定的字段值，对剩余每一行跑一次比较，留下的写进下一个缓冲（双缓冲 ping-pong）。评分列（如 Float Difference）不剔除，而是给每行累加一个归一化代价。
3. 若剩下多行且有评分，按代价从小到大排序。
4. 依次取命中行：先把所有输出列的值写回上下文，再调该行的结果回调；回调返回「停」就取第一个结果，返回「继续/失败」就接着试下一行。没有任何一行成功，才落到 FallbackResult。

读字段的核心是「属性链解析」：每个绑定存一条 `FName` 链，编译期展开成偏移量/函数链，运行期从上下文对象一路解到最终字段的地址再读/写。这也是它既能读对象字段、也能读结构体字段、还能调函数取值的缘故。Gameplay Tag 列（`FGameplayTagColumn`）在这之上多一层：它比较的是 `FGameplayTagContainer`，支持「输入包含行标签」还是「行包含输入标签」两种方向、精确/模糊匹配，以及反选。

## 和谁配合

- **AnimNode_ChooserPlayer**：动画侧的实际玩家。它内部就是一个 BlendStack，挂一个 `Chooser`（一个 ObjectChooserBase，可以是「评估 Chooser 表」或「查 Proxy 表」），按 `EvaluationFrequency`（初始更新 / 变相关 / 循环 / 每帧）调用评估，拿到资产后 BlendTo 播放，并支持每资产 StartTime / 镜像 / 曲线覆盖等设置。Chooser 只负责「选」，播放和混合归它。
- **Motion Matching / PoseSearch**：典型用法是在 PoseSearch 查询前，用一张 Chooser 按移动模式、姿态、是否持械等条件挑出「允许的 Database 集合」，再把集合交给 PoseSearch 做真正的姿势搜索。开 `bStartFromMatchingPose` 时，ChooserPlayer 会先把 Chooser 选出的所有资产喂给 PoseSearch，由它挑最贴的姿势和起始时间。
- **Proxy Table**：和 Chooser 是两种互补的决策。Chooser 是「按条件选出结果」；Proxy Table 是「按 GUID 查表替换」——一个 Proxy Asset 是间接引用（带 Type + Guid），`FLookupProxy` 拿这个 Guid 去 Proxy Table 里查出该关卡/该角色实际对应的资产（支持 `InheritEntriesFrom` 做层级覆盖）。所以 **Chooser 偏条件选择，Proxy 偏资源替换**；两者都实现 ObjectChooserBase，可以互相嵌套：Chooser 的结果可以是 Proxy 查询，Proxy 的值也可以是再评估一张 Chooser。

## 常见坑

- **输入状态与查询不同步 → 滞后一帧或抖动**：`EvaluationFrequency` 决定 Chooser 多久重评一次；非 OnUpdate 模式下，选中结果会被缓存（`UpdateAssetPlayer` 里只有到点才重选，否则一直播当前资产）。如果上下文字段（移动模式、速度、GameplayTag）由玩法代码在别的相位写入，读到的就是上一帧的值；两个状态在阈值附近来回跳时，就可能一帧 A 一帧 B 地抖。要么把 Chooser 设为 OnUpdate，要么让写入侧与动画查询对齐相位，要么在阈值处做迟滞。
- **上下文类型对不上**：列绑定的是属性链，链的前缀必须落在表声明过的 ContextData 里的某个类/结构体上。类改名、字段改名、或评估时上下文对象里根本没有那个类，属性链解析失败，列会走「透传」——表现为「这列好像没起作用」，而不是报错。
- **字段读不到 = 静默透传**：过滤列的 `Filter` 在取不到值时直接原样放行（`IndexListOut = IndexListIn`）。这有利于编辑期联调，但上线后一旦字段绑错，这列就变成透明列，行不再被它筛掉。排查「规则失效」先查列的属性绑定，以及评估时上下文里是否真塞进了那个对象。
- **兜底没配 → 返回 null**：所有行都不命中且没配 FallbackResult，返回空。下游拿到空资产会怎样取决于节点：ChooserPlayer 里拿不到资产就停在旧资产或空转。
- **它不混合也不算 IK**：别指望 Chooser 表直接产出平滑姿势。选出的资产之间的过渡是 ChooserPlayer / BlendStack 在做；姿势上的方向修正、脚部落地是 Orientation Warping / Leg IK 的活。Chooser 只决定「允许谁」。

## 常见问题

- **Chooser 和 Proxy Table 到底怎么选？** 需要「按运行时状态筛出一类资源」用 Chooser；需要「同一个占位资产在不同角色/关卡里换不同实体」用 Proxy Table。二者可以嵌套，不必二选一。
- **Chooser 能选姿势吗？** 不能。它选资源；姿势选择交给 PoseSearch / Motion Matching。它最常见的作用是把 PoseSearch 的 Database 候选集先缩小。
- **为什么我加的条件列不生效？** 先查列的属性绑定是否落在声明过的上下文类型上、评估时上下文里是否真有那个对象；再确认该列没被禁用（编辑态禁用列会在求值时跳过）。读不到值时列会透传而不是报错，所以「没反应」往往不是真的没运行。
- **什么时候用 OnUpdate、什么时候用 OnLoop？** 选择结果要随状态即时变化（如切换 Database）用 OnUpdate；结果天然按「一段动画播完再换」用 OnLoop，能少评很多次、也更稳。

## 参考资料

- [Dynamic Asset Selection in Unreal Engine（Chooser / Proxy Table 官方文档）](https://dev.epicgames.com/documentation/unreal-engine/dynamic-asset-selection-in-unreal-engine)
- [Game Animation Sample Project](https://dev.epicgames.com/documentation/en-us/unreal-engine/game-animation-sample-project-in-unreal-engine)
- 引擎源码（UE 5.8）：
  - `Engine/Plugins/Chooser/Source/Chooser/Public/Chooser.h`、`Private/Chooser.cpp`（`UChooserTable` 与 `EvaluateChooser` 求值流程）
  - `.../Chooser/Public/IChooserColumn.h`、`IObjectChooser.h`、`IHasContext.h`、`ChooserPropertyAccess.h`（列接口、结果基类、上下文与属性链）
  - `.../Chooser/Internal/AnimNode_ChooserPlayer.h`、`Private/AnimNode_ChooserPlayer.cpp`（动画播放器与评估频率）
  - `Engine/Plugins/Chooser/Source/ProxyTable/Public/ProxyTable.h`、`ProxyAsset.h`、`Private/LookupProxy.cpp`（Proxy Table / Proxy Asset 与查表替换）
