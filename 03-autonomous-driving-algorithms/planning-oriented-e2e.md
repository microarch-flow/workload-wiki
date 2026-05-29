# Planning-oriented E2E

上级：[Autonomous Driving Algorithms](README.md)
相关：[BEV Perception](../02-vision-and-3d-perception/bev-perception.md), [E2E Workload](../06-chip-workload-analysis/e2e-workload.md)

## 这页在回答什么问题

这页回答 planning-oriented E2E 为什么成为自动驾驶算法路线的重要节点。它不是简单把 perception、prediction、planning 串成一个大模型，而是用统一 BEV / query / token 表示，把多个任务和最终 planning loss 联合优化。

## 典型结构

```text
multi-view cameras / lidar
   ->
image or point encoder
   ->
BEV / occupancy / vectorized scene representation
   ->
detection + map + motion + occupancy + planning heads
   ->
trajectory loss + auxiliary task losses
```

代表路线包括 UniAD 和 VAD。UniAD 把 detection、tracking、mapping、motion forecasting、occupancy prediction 和 planning 放进统一框架；VAD 强调 vectorized scene representation，用稀疏向量替代过度 dense 的表示，服务于规划。

## 为什么叫 planning-oriented

传统多任务感知模型可能每个 head 都很强，但最终规划并不一定收益最大。Planning-oriented E2E 的训练目标会把规划质量作为核心评价，使中间表示服务于 ego action，而不是只服务于 perception benchmark。

这带来两个变化：

- 中间任务从“独立输出”变成“对 planning 有用的监督”。
- 模型接口从 object list 扩展为 BEV feature、query state、vector map、future occupancy 等可微表示。

## 关键设计维度

| 维度 | 选择 | 影响 |
| --- | --- | --- |
| 表示 | dense BEV、occupancy、vectorized scene、query set | 决定 memory footprint 和 head 结构 |
| 任务 | detection、map、tracking、prediction、planning | 决定监督复杂度和训练稳定性 |
| 时序 | short history、long history、state cache | 决定 temporal cache 与 latency |
| 规划 | trajectory regression、candidate scoring、cost map | 决定输出可控性和安全约束接口 |

## 工程边界

Planning-oriented E2E 并不意味着取消安全壳。量产系统通常仍会保留 rule check、collision check、comfort constraint、fallback planner 和监控模块。E2E 的价值在于生成更一致的 scene/action representation，而不是替代所有系统安全机制。

## 成熟度判断

- 成熟：BEV 多任务模型、trajectory planning head、nuScenes/nuPlan 等离线评测。
- 发展中：闭环规划指标、长尾场景泛化、vectorized scene representation。
- 前沿：把 VLM/VLA reasoning、World Model rollout 和 planning-oriented E2E 融合为统一 policy。

## 一句话理解

Planning-oriented E2E 的核心是让感知表征从服务检测指标，转向服务最终驾驶轨迹；它把自动驾驶 workload 从单任务 backbone 推向共享表征、多 head、时序状态和规划闭环。

## Workload Characterization

- 计算密度：multi-view encoder、BEV transformer、query decoder、multi-task heads 共同构成主计算；planning head 通常小于 scene encoder。
- 访存模式：BEV feature、query state、map vector、occupancy grid 和历史帧 cache 需要跨 head 复用，读写路径复杂。
- 并行性：camera view、BEV cell、query、task head 可并行；tracking、motion 和 planning 之间存在依赖。
- 数据复用：共享 BEV / query feature 是关键收益；辅助任务越多，feature reuse 越强，但中间 activation 也越大。
- 量化敏感度：backbone 和部分 MLP head 可低比特；geometry transform、attention score、trajectory regression 和 collision margin 对误差敏感。
- 瓶颈类型：训练侧是多任务 loss 与大 activation；推理侧是 BEV construction、temporal cache 和 decoder latency。
- 对硬件的核心需求：高效 multi-view feature fusion、attention/query decode、BEV cache 管理、低 batch 推理、多 head 并发调度。

## 参考来源

- Hu et al., `Planning-oriented Autonomous Driving`, CVPR 2023 / arXiv:2212.10156，UniAD，成熟度：研究成熟，查证日期：2026-05-29。
- Jiang et al., `VAD: Vectorized Scene Representation for Efficient Autonomous Driving`, ICCV 2023 / arXiv:2303.12077，成熟度：研究成熟，查证日期：2026-05-29。
- Caesar et al., `nuPlan: A closed-loop ML-based planning benchmark for autonomous vehicles`, CVPR ADP3 2022 / arXiv:2106.11810，成熟度：常用评测基准，查证日期：2026-05-29。
- Dauner et al., `NAVSIM: Data-Driven Non-Reactive Autonomous Vehicle Simulation and Benchmarking`, NeurIPS 2024 Datasets and Benchmarks / arXiv:2406.15349，成熟度：新兴评测基准，查证日期：2026-05-29。
