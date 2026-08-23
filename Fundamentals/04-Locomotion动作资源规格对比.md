# Locomotion 动作资源的规格差异

> 三种系统对「起步几步、循环几拍、几向、脚相怎么对齐、要不要标定速度」的约定不同。ALS 与 Kai 用**离散片段目录**，Motion Matching 用**连续动捕数据库**。

## 解决的问题

给一套新动作集做接入或重定向前，除了根运动（见 [03](./03-动画资产的根运动规格.md)），还要先弄清另一条边界：**移动事件是「离散片段」还是「连续采样」**。

- 离散片段：起步、循环、停止、转身各是一段独立的、有固定相位和步数的动画，由状态机选择。
- 连续数据库：把大量连续动作捕捉 clip 采样进 PoseSearch Database，运行时按轨迹匹配选出姿态，事件之间没有固定边界。

这两条路线对资产的要求几乎处处相反。本篇按「起步、循环、停止、转身、方向、脚相、配套数据、数量级」逐项对比。

## 起步：几步？固定还是连续？

「起步需要几步」是最能暴露分歧的问题：

| 系统 | 起步资产 | 步数语义 |
| --- | --- | --- |
| ALS | `Acceleration` 前/后/左/右 4 段 | 独立短加速段，只覆盖从静止蹬地到进入循环节奏的过渡（典型 1~2 步） |
| Kai | 四向平移起步 + 可选 90°/180°旋转起步 | 独立起步段；`FirstStepAlpha` 把动画时间 `[0.1, 0.4]` 映射为「第一步是否已建立」；起步从动画开头（累计位移约 `1` 单位处）起播，由逐帧距离匹配追上真实位移 |
| Motion Matching | 无独立起步资产 | 起步是任意连续 clip 里「速度 0 → 加速」的片段，由轨迹匹配选中，步数不固定 |

Kai 的 `FirstStepAlpha` 最能说明离散方案的语义：起步 clip 用动画时间明确标记出**第一步落地**的锚点（`FirstStepAlpha` 从 0 到 1），而「距离 1」只是起步的起播点，约等于动画开头。ALS 的 `Acceleration` 同理，是覆盖起步前 1~2 步的短段。Motion Matching 则根本不要求存在「一段叫起步的动画」——它可能匹配到某个 10 米直线行走 clip 的前 3 步，也可能匹配到某个变向 clip 的加速段。

## 循环：完整步态周期，还是连续覆盖

- **ALS**：循环是 6 向（前/后/左前/左后/右前/右后）× Walk/Run，每向一个**可循环 clip**（一个完整步态周期 = 左右各一步），再进 Blend Space 按速度混合，并用 `StrideBlendAmount` 调步幅。
- **Kai**：四向（前/右/左/后）锚点 + `FwdLeanLeft/Right`，每向一个可循环 clip；四向序列必须「对齐左右脚相位、循环起点、单周期时长、平均根位移速度和重心高度」，扇区内角度靠 Orientation Warping 补齐。
- **Motion Matching**：不需要严格的循环点。GASP 的 Walk/Run/Sprint 各几百段连续 clip（直线 `Neutral`、弧线 `Arc`、折返 `Box`、`Diamond`、绕圈 `Circle_Strafe`、过渡 `Run_to_Walk` 等），每段以 `Lfoot`/`Rfoot` 起止；循环感来自数据库在连续轨迹上的采样，而非一段 clip 反复重播。

## 停止与转身

- **停止**：ALS 是 `Stop_Left`/`Stop_Right` 左右脚两段；Kai 是四向 × 步态 × 左右脚（`UsePerFootStop`），并按速度卡迪纳尔方向选资源；Motion Matching 无独立停止 clip，靠连续 clip 的减速段。
- **转身**：ALS 用 `TurnInPlace` 90°/180° × 左右 × 站/蹲（8 段）+ `RotateInPlace`（小角度靠 RotationYawOffset 曲线）；Kai 用 TIP 90° 左右（站立选择器当前只返回 90，180° 虽在资源目录但未核实被该路径使用）+ Pivot 四向 + 旋转起步；Motion Matching 靠 Arc/Box/Diamond/Circle 这些转向 clip 天然覆盖，不设独立转身资产。

## 方向表达：连续混合 vs 离散锚点

- **ALS**：Blend Space 连续混合 + `RotationYawOffset` 曲线修正身体朝向。
- **Kai**：四向离散锚点（先选一个最适合的资源），再用 Orientation Warping 补足锚点之间的角度残差。
- **Motion Matching**：数据库轨迹连续匹配，clip 密集覆盖方向空间（不是「几条 + 插值」，而是「几百条里查最像的那条」）。

## 标定速度与脚相

- **标定速度**：ALS 要求动画按标定速度制作（Walk 150 / Run 350 / Sprint 600 cm/s），`PlayRate = 实际速度 ÷ 标定速度`；Kai 要求序列的根运动平均速度接近该步态目标速度，`PlayRateClamp` 只吸收小偏差；Motion Matching 不需要标定速度，根运动速度逐采样直接读。
- **脚相**：ALS 用 `B_Als_AnimationModifier_FootSyncMarkers` 自动生成脚步同步标记 + Foot Lock；Kai 用 `KLSLeftFootUp` 曲线选左右脚版本 + 同步标记 Modifier；Motion Matching 用 `Lfoot`/`Rfoot` 双版本 clip 保证过渡脚相一致。

## 配套数据

- **ALS**：`RotationYawOffset` 曲线、Stride Blend 曲线、Foot Lock 曲线。
- **Kai**：距离/旋转曲线（编辑器从根运动缓存）+ 命名曲线（`KLSSpeed`、`KLSGaitAlpha`、`KLSLeftFootUp` 等）+ Mask 曲线。
- **Motion Matching**：PoseSearch Database（Schema、轨迹通道、采样密度），外加少量曲线供 Chooser / Foot Placement。

## 数量级

| 系统 | 规模 |
| --- | --- |
| ALS | 几十段离散 clip（起步 4 向、循环 6 向 × 2 步态、停止 2、TIP 8 …） |
| Kai | 每步态每方向一套（站立 + 蹲伏），中等规模 |
| Motion Matching | 数百段连续 clip，仅 Walk/Run/Sprint 就约 353 + 360 + 65 = 778 段 |

## 为什么走向两条路线

离散片段适合**状态机 + 明确事件**：起步、停止、转身是有限个离散事件，各自做成独立 clip，状态机负责选择，代码负责距离/相位对齐。连续数据库适合**轨迹查询**：「下一步该怎么动」交给数据库按轨迹查，因此需要 clip 密集覆盖运动空间，于是资产多、且不按事件切分。

这也解释了验收口径的不同：离散方案验收「每段 clip 的相位、步数和方向是否对齐」，连续方案验收「clip 是否足够覆盖速度/方向/弧度空间、根运动轨迹是否真实」。

## 常见误解

- **「起步都是一段固定动画」**：Motion Matching 没有固定起步片段。
- **「循环都要做成可循环的 2 步周期」**：只对 ALS/Kai 成立；Motion Matching 靠连续 clip 采样，不要求严格循环点。
- **「四向资源不够就多插几条」**：ALS 用 Blend Space 连续插值，Kai 用离散锚点 + Warping；两家的「方向补全」机制不同，不能套用同一套资产。
- **「转身都要 90/180 四段」**：ALS 是，Kai 站立只走 90 左右，Motion Matching 靠转向 clip 覆盖。

## 相关主题

- [动画资产的根运动规格](./03-动画资产的根运动规格.md)
- [ALS：动画资产组织](../ALS/09-动画资产组织.md)
- [Kai：起步——靶向与非靶向](../KAI/04_起步_靶向与非靶向的动画选择.md)
- [Kai：循环——用速度维持步频](../KAI/05_循环_步态与方向的移动表现.md)
- [Kai：动画数据、曲线与制作约定](../KAI/16_动画数据_曲线与制作约定.md)
- [Motion Matching：系统架构与数据流](../MotionMatching/02-系统架构与数据流.md)

## 参考资料

- ALS-Refactored 资产目录：`C:\AnimationExample\Plugins\ALS-Refactored-4.17\Content\ALS\Animations\`（`Grounded\Acceleration`、`Grounded\WalkRun`、`TurnInPlace`、`RotateInPlace`；`Stop_Left/Right`）
- ALS-Refactored 标定速度：`Source\ALS\Public\Settings\AlsStandingSettings.h:12-19`（`AnimatedWalkSpeed` 150 / `AnimatedRunSpeed` 350 / `AnimatedSprintSpeed` 600）
- Kai 起步/循环/停止/Pivot/TIP：仓库内 `KAI\04`、`05`、`06`、`07`、`08`、`16` 笔记（来源为 Kai Locomotion 源码，本机路径已不可用）
- GASP 连续 clip 目录：`C:\AnimationExample\Content\Characters\UEFN_Mannequin\Animations\{Walk,Run,Sprint}\`（Walk 353 / Run 360 / Sprint 65，含 Arc/Box/Diamond/Circle 与 `Lfoot`/`Rfoot` 变体）
- 引擎官方文档：[Game Animation Sample Project](https://dev.epicgames.com/documentation/en-us/unreal-engine/game-animation-sample-project-in-unreal-engine)
