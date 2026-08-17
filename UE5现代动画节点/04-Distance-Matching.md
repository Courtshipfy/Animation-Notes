# Distance Matching：按距离推进动画

> Distance Matching 把“动画现在应播放到哪一帧”从固定时间问题改成距离查询问题。

## 解决的问题

停止、Pivot、起步和落地的进入速度不完全相同。若固定按时间播放一段动画，角色可能已经停下而动画还在迈步，或还没到落地点就提前进入落地姿势。仅调播放速率也会改变动作节奏。

Distance Matching 让动画资产携带 Distance Curve；运行时根据剩余距离、已移动距离或到地面距离在曲线上反查动画时间。通常由 Animation Locomotion Library 的节点函数更新 Sequence Evaluator 的显式时间。

```text
移动模型给出距离 D
       │
Distance Curve：时间 t → 距离 d
       │ 反查 d ≈ D
Sequence Evaluator 播放到时间 t
```

## 三个必须对齐的部分

1. **动画曲线**：曲线必须随动画的实际根位移或所定义的测距方向变化，且压缩设置允许运行时读取。
2. **距离来源**：停止一般使用预测制动距离，Pivot 使用到转向点的距离，落地使用到地面的距离；不要混用单位、轴向或正负方向。
3. **显式时间控制**：Sequence Evaluator 不再单纯随 Delta Time 累加，而由曲线查询结果驱动；没有正确写入显式时间，曲线再正确也不会生效。

## 常用场景与边界

| 场景 | 查询的距离 | 目的 |
| --- | --- | --- |
| 停止 | 剩余制动距离 | 让停步阶段对应真正的减速位置 |
| Pivot | 到转向点或已越过的距离 | 让蹬地、转向和反向移动衔接 |
| 落地 | 到地面的高度 | 在接触前进入合适的准备或缓冲姿势 |
| 起步 | 已推进的距离 | 让启动阶段与实际位移同步 |

它不负责预测距离，也不选择动画。Character Movement、轨迹预测或玩法代码负责给出可信的距离；状态机、Chooser 或 Motion Matching 负责决定当前使用的动作。距离模型与动画根位移不一致时，Distance Matching 只会把这种不一致稳定地放大。

## 常见问题

- 曲线没有启用可运行时索引的压缩设置，节点函数无法正确读取数值。
- 输入距离在停止前后跨过零点，曲线方向却按另一种符号定义，导致动画倒跳或卡在端点。
- 同时用大幅 Play Rate 修正和 Distance Matching 推进同一段动画，节奏容易失去可控性。
- 距离超出曲线有效范围时没有定义端点策略，造成姿势冻结或突然跳转。

## 相关主题

- [根运动与根骨修正](./03-根运动与根骨修正.md)
- [ALS 起步、停止与转身](../ALS/06-起步停止与转身.md)

## 参考资料

- [Distance Matching](https://dev.epicgames.com/documentation/en-us/unreal-engine/distance-matching-in-unreal-engine)
- [Animation Locomotion Library](https://dev.epicgames.com/documentation/en-us/unreal-engine/API/Plugins/AnimationLocomotionLibrary)
