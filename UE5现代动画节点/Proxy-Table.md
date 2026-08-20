# Proxy Table（代理表）详解

> Proxy Table 做的事一句话就能说清：把「引用某个资源」从「写死一个具体资产」变成「写死一个插槽名，运行时再按上下文解析成真正该用的那个资产」。调用处引用的是 Proxy（代理），真正决定用哪个动画数据库、Montage 或别的对象的，是运行时去查的那张表。它的聪明之处在于，被替换的「值」本身可以再是一个选择器，于是「同样是 Run Database」这件事，能按体型、武器、风格层层落到不同版本。

## 它解决的问题

想象一个多角色项目：几十个角色共用同一套 AnimBP / 动画图，但每个角色的 Motion Matching 数据库、Pose Search 数据库、技能 Montage 都不一样。如果动画图里直接硬引用「人类男性的数据库」，那每加一个体型、一种武器、一套风格，你就得复制一份动画图，或者塞进一堆分支节点——资产一变，图就变，维护成本爆炸。

Proxy Table 要拆开的正是这层耦合：**动画图里不再引用具体资产，只引用一个「语义槽位」**（比如「Run 数据库」「攻击 Montage」）。这个槽位是一个 `UProxyAsset`，它自己不指向任何动画，只是一个带类型声明的占位符。真正的「槽位 → 具体资产」映射集中写在一张 `UProxyTable` 里，每个角色挂自己的表，运行时按上下文查出来。

它明确**不做**的事，理解了边界才不会误用：

- **不做条件判断**：它不回答「现在是走还是跑」。这个槽位此刻该填哪个资产，是交给槽位值里的 Chooser（或直接引用）去决定的；Proxy Table 自己只负责「按 key 找到这一行」。
- **不要求资产变体**：如果某个资源永远不随上下文变，它不会带来任何好处，反而多一层间接（见「什么时候别用」）。
- **不是数据驱动的万能解耦**：它解决的是「同一张图、多个角色/多套变体」的资源替换，不是动画状态机逻辑本身。

## 两个资产、一张映射：核心模型

整个系统只有两类资产，理解它们的分工就理解了 Proxy Table：

| 资产 | 职责 | 关键内容 |
|---|---|---|
| `UProxyAsset` | 占位符，给槽位一个**稳定的身份** | 一个 `Guid`（身份/键）、一个 `Type`（声明解析结果必须是什么类，供编辑器过滤和类型检查）、一组 `ContextData`（声明评估这一行需要哪些上下文参数） |
| `UProxyTable` | 映射表，把身份翻译成资产 | 若干条目：`Proxy`（键）→ 一个「值」 |

这里的「值」是设计上最关键的一步：**它不是一个普通的对象指针，而是一个 Object Chooser**（`FObjectChooserBase` 的实例化结构体）。这意味着每个槽位的值可以是：

- **直接引用**：一个写死的 `UObject`（最朴素的「代理 → 资产」）。
- **一个 Chooser 表**：按上下文条件（体型、武器、风格……）再选一次。这就是「同样是 Run Database，不同体型用不同版本」的实现载体。
- **再套一个 LookupProxy**：继续做下一层间接引用，可以串成链。

键是 `ProxyAsset` 的 **Guid，而不是资产指针**。这是刻意为之：调用处引用 Proxy 资产，运行时拿它的 Guid 去表里二分查找，命中后把那一行的「值」评估成最终对象。因为键是 Guid 而不是路径，Proxy 资产改名、移动不会破坏映射；反过来，Guid 一旦撞了（比如在编辑器外复制了 Proxy 资产），两行就会在构建时被当成冲突。

> 早期版本用 `FName` 做键（`FProxyEntry::Key`），现已弃用，只留作旧内容的兼容路径。新内容一律走 `UProxyAsset` + Guid。

## 关键流程：一次解析是怎么发生的

把「动画图里的一个 Proxy 引用」走到「最终资产」，分三段。

### 1. 调用点：一个 LookupProxy 占住槽位

动画图（或 Blueprint）里放的是 **Evaluate Proxy 节点**。它编译后产出的不是一个直接函数调用，而是一个 `FLookupProxy` 结构：里面存着 `Proxy`（哪个槽位）和一个 **ProxyTable 参数绑定**（从上下文对象身上读出该用哪张表）。

也就是说，调用方只声明两件事：**「我要哪个槽位」和「表从哪来」**。表从哪来，通常就是角色/上下文对象身上的一个 `UProxyTable*` 属性——每个角色指向自己的表，所以同一张动画图能服务所有角色。节点也可以显式接一张表（Override Table），用于局部覆盖。

### 2. 运行时：Guid → 值 → 评估

`FindProxyObject(Key, Context)` 的流程很直白：

1. 用 Proxy 的 Guid 在排序好的键数组里**二分查找**，找到对应行；
2. 先写出这一行的**结构输出**（`OutputStructData`，把额外的结构体数据拷进上下文的绑定属性，作为伴随结果的副作用数据）；
3. 把这一行的「值」当成 Object Chooser 调用 `ChooseObject(Context)`，得到最终对象；找不到行、或值不是 Chooser，就返回空。

关键点在于：**评估「值」用的就是同一个 Context**。所以值里的 Chooser 表能读到角色当前的全部上下文，去做「这个角色用哪套数据库」的二级选择。

### 3. 表的构建：编辑器条目 → 运行时扁平数组

编辑时，表是一组 `Entries`（每条含 Proxy、值、结构输出）加一个 `InheritEntriesFrom`（继承哪些父表）。这形态方便人维护，但不适合频繁二分查找。所以在 `PostLoad` / 事务之后会 `BuildRuntimeData()`，把条目**扁平化、按 Guid 排序**成两个平行数组（键数组 + 值数组），并编译好属性绑定，运行时只做一次二分查找。

构建时的两个细节值得记住：

- **继承合并，先到先得**：先收自身条目，再递归收父表条目；Guid 相同的只保留先遇到的那个，所以**子表条目覆盖父表**。编辑器中继承来的行是只读的，单独标出「Inherited from」。
- **Guid 冲突会打 Error**：两个不同的 Proxy 资产若 Guid 相同（典型是编辑器外复制导致），构建时会报错——但运行时仍只有一个胜出，表现为「改了这个却影响那个」。

## 和 Chooser 的边界

Proxy Table 和 Chooser 常被一起提到，因为它们出自同一个插件、又天然互补。一句话分清楚：

- **Chooser 偏「条件选择」**：给定上下文，逐行判断条件，选出命中的行，返回那一行的结果。
- **Proxy 偏「资源替换」**：给定一个槽位身份（Guid），找到「这个槽位在本次上下文中该填哪个资产」。

两者的组合方式很自然：**Proxy Table 的「值」里放一个 Chooser 表**。Proxy 负责「哪个槽位」，Chooser 负责「槽位里的哪个变体」。反过来，Chooser 表的一行结果也可以是一个 LookupProxy——Chooser 先按条件选出「用哪个槽位」，再交给 Proxy 解析。谁在外层、谁在内层，取决于你的语义更接近「先选资源类别」还是「先选槽位」。

区分它们还有个实用角度：**Chooser 表本身是资产，和结果资产强绑定；Proxy 表则是「身份 → 值」的字典，值可以是任意 Chooser**。所以 Proxy 表天然适合做「每个角色一份」的覆盖层，而 Chooser 表更多是「全局规则」的载体。

## 典型应用：多角色共用一张动画图

落地到动画项目，最常见的形态是这样：

1. 动画图里，所有「角色相关」的资产引用（Run/Idle 的 Pose Search 或 Motion Matching 数据库、技能 Montage、风格化叠加动画……）全部换成 **Proxy 资产**，图的逻辑对所有角色完全一致。
2. 为每个语义槽位建一个 Proxy 资产，并声明它的 `Type`（比如数据库类、`AnimMontage`）和需要的 `ContextData`。
3. 每个角色建一张 Proxy Table：把自己的 Proxy 资产映射到自己的数据库/Montage。角色身上挂这张表。
4. 如果某个槽位要按武器/体型再分版本，把那一行的「值」写成一个 Chooser 表，表里按上下文选具体资产。

效果是：**加一个新角色 = 复制一张 Proxy Table 改几个映射**，动画图一行不动；「同样是 Run Database，重装和轻装用不同版本」这种规则也集中在 Proxy Table 的某一个槽位里，而不是散落在动画图各处。

这套机制对应引擎里的「Evaluate Proxy」节点（菜单分类就在 Animation 下），运行时走的是 `FLookupProxy → UProxyTable::FindProxyObject` 这条路径；节点只是编辑期的可视化入口，真正干活的是运行时模块里的这段解析。

## 什么时候别用

Proxy Table 是一层间接引用，**间接引用只在「同一个槽位会换成不同东西」时才有价值**：

- 如果某个数据库、Montage 永远只有一份，直接引用更直白——少一层表、少一次二分查找、少一个可能配错的地方。加了 Proxy 反而让「这个槽位到底指向谁」需要跳两张表才能看清。
- 如果变体规则很浅（就一两个角色、两三种情况），直接在动画图里接 Chooser 或分支往往更易读，不必为了「统一」提前上 Proxy。

一句话：**资源随上下文/角色变 → 用 Proxy Table；资源恒定 → 直接引用。**

## 常见问题

- **Evaluate Proxy 节点返回空**：优先查两层——① 上下文对象有没有正确提供 `UProxyTable`（属性绑定读不到表、或表为 null，解析直接落空）；② 这张表里到底有没有给这个 Proxy 资产建条目，没建就是二分查找未命中返回空。注意：**Proxy 资产本身不包含任何映射**（基类的 `FindProxyObject` 直接返回空），光有 Proxy、没有表，什么也解析不出来。
- **两个 Proxy 的表现串了 / 改 A 影响 B**：几乎都是 **Guid 撞车**。正常在编辑器内复制 Proxy 资产会自动生成新 Guid（`PostDuplicate` 里 `NewGuid`），但在编辑器外复制、或用了旧 FName 键的资产，就可能撞。构建时日志会打 Error，别忽略。
- **结构输出没生效**：这一行的 `OutputStructData` 会把值拷进上下文里绑定的属性，但**只有目标结构体类型完全一致才拷贝**，类型不匹配会打 Warning 然后跳过。先对一下 Proxy 的 ContextData 声明和实际传入的 struct 是不是同一个。
- **「为什么我改了父表，子角色没变」**：继承是「先到先得」，子表自己的条目会**覆盖**父表同名 Guid 的条目。你以为在继承，其实是子表已经有一行盖住了。
- **用到了 `EvaluateProxyTable`（FName 版）**：这是明确的**临时兼容函数**，官方注释写着「please switch to EvaluateProxyAsset」。新内容统一用 Evaluate Proxy 节点 / `EvaluateProxyAsset`，别再按 FName 键手工哈希。

## 参考资料

- [Dynamic Asset Selection（动态资产选择）—— 官方主文档，Proxy Table / Chooser 的入口与工作流](https://dev.epicgames.com/documentation/unreal-engine/dynamic-asset-selection-in-unreal-engine)
- [ProxyTable API 参考](https://dev.epicgames.com/documentation/unreal-engine/API/Plugins/ProxyTable)
- 引擎源码（UE 5.8）：
  - `Engine/Plugins/Chooser/Source/ProxyTable/Public/ProxyTable.h` 与 `Private/ProxyTable.cpp`（`UProxyTable` 条目结构、`BuildRuntimeData` 的继承合并、`FindProxyObject` 的二分查找与值评估）
  - `Engine/Plugins/Chooser/Source/ProxyTable/Public/ProxyAsset.h` 与 `Private/ProxyAsset.cpp`（`UProxyAsset` 占位符：Guid、Type、ContextData；Guid 生成与冲突来源）
  - `Engine/Plugins/Chooser/Source/ProxyTable/Internal/LookupProxy.h` 与 `Private/LookupProxy.cpp`（`FLookupProxy` 如何从上下文取表并解析）
  - `Engine/Plugins/Chooser/Source/ProxyTableUncooked/Private/EvaluateProxyNode.cpp`（「Evaluate Proxy」蓝图节点的编译展开）
