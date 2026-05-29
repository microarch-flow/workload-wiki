# Occupancy Workload

上级：[Chip Workload Analysis](README.md)
相关：[Occupancy Prediction](../02-vision-and-3d-perception/occupancy-prediction.md), [BEV Workload](bev-workload.md)

## 这页在回答什么问题

这页分析 occupancy 预测的芯片压力。occupancy 把场景表示从 2D BEV 平面推到 3D 体素（每个 voxel 标注 occupied/free/unknown 或带语义类别），核心压力随之从"平面特征"变成"3D/4D 张量的容量、稀疏访问和未来时序"。它和 BEV 共享 view transform 的不规则访问问题，但又多了一层 3D 容量爆炸。

## 容量为什么是 occupancy 的第一约束

先算一笔账。一个 `200 × 200 × 16` 的体素栅格有 64 万个 cell。如果每个 cell 带 C 个语义/特征通道（比如 16），FP16 存储就是 `640000 × 16 × 2 B ≈ 20 MB`——这还只是一帧的特征。再叠加多个 decoder 中间层、历史帧、以及预测未来若干步的 horizon，activation 和 logits 很快就超过任何现实的片上 SRAM 容量。这就是为什么 occupancy 的第一瓶颈通常是 **memory-capacity-bound**，而不是算力。dense 3D conv 的算力虽然也不低，但真正先撑爆的是容量。

## dense 与 sparse 的取舍：一个 determinism 难题

面对容量压力，自然的应对是稀疏化——真实世界绝大部分体素是空的，只存活跃 voxel 能省下大量计算和存储。但这对 deterministic NPU 是个两难。

dense occupancy 接口简单：规则 3D 张量，可以像 2D feature map 一样 3D tiling，访问可预测、延迟有界，代价是容量和带宽压力大。sparse occupancy（sparse conv、rulebook 索引）更贴合真实占用分布，计算量可能降一个数量级，但活跃 voxel 的分布随场景动态变化，需要 metadata 解码、indice pair、rulebook 查找和负载均衡——这些都是动态控制流，会带来 **control-overhead-bound** 的风险，且 p99 latency 难以保证。对一颗追求 bounded latency 的车端 NPU 来说，"稀疏省下的算力"可能被"不规则访问和动态调度的开销"吃掉。

因此 occupancy 的架构建模不能只看 dense 3D FLOPs，必须同时记录有效稀疏率、活跃 voxel 的空间分布、future horizon 长度，以及 planner 对 occupancy 的查询模式（是全量读还是沿轨迹稀疏查询）。

## 时序与查询：horizon 和 collision check

occupancy 越来越多地预测未来若干步的占用演化（如 OccWorld 这类 4D occupancy world model），这把状态压力再乘上 horizon，cache 容量随预测步数增长。下游 planner 通常不需要读全部体素，而是沿候选轨迹做稀疏的 collision query——这是随机、低延迟、小粒度的访问。如果 occupancy logits 在 NPU、而 collision check 在 CPU，logits 的来回搬运会形成同步瓶颈，这是 occupancy 集成进 E2E 系统时的隐藏延迟来源。

## 可建模参数

`voxel resolution`（如 200×200×16）直接决定容量与 3D conv 算力；`semantic channels` 放大每个 voxel 的存储；`effective sparsity`（有效稀疏率）决定 sparse 路线的真实收益；`temporal horizon` 决定未来占用的状态驻留；`query pattern`（dense 全读 vs 沿轨迹稀疏查询）决定 planner 接口的访问形态。

## 硬件连接

RAM：dense grid 的容量是第一问题，sparse metadata 会进一步降低 row locality（见 RAM wiki）。DMA：需要 3D tile 搬运、sparse block 搬运，以及 metadata 与 feature 的协同搬运（见 DMA wiki）。NOC：3D tile 间的 halo exchange、sparse block 分发、query 结果汇聚都会产生跨 tile 通信（见 NOC wiki）。CIM：局部 dense 3D conv 可用 CIM，但 sparse metadata 解码和动态 query 不适合。PCIE/host：collision query 和 planner 若在 CPU，occupancy logits 的往返会成为同步瓶颈，应尽量让 query 在 NPU 近端完成。

## archax 建模

Resource：voxel cache 容量（最关键）、3D conv TOPS、sparse metadata buffer、DRAM bandwidth。Topology：3D grid tile 到 NPU tile 的映射，以及 planner/query 与 occupancy memory 的近端连接。Interaction：`BEV/voxel feature → occupancy decode → future cache → trajectory query` 的链路。Capability：3D conv、sparse conv、indexed query、metadata decode、multi-resolution tile，以及 INT8 logits 配合高精度几何。archax 探索时应把 dense/sparse 作为两个不同工作点分别评估——它们一个是 capacity-bound、一个是 control-overhead-bound，不能合并成一个平均画像。

## 一句话理解

occupancy 是把感知推到 3D 体素后引爆容量的 workload：dense 路线 capacity-bound、sparse 路线 control-overhead-bound，架构选择本质是在"放得下"和"调得动"之间权衡。

## Workload Characterization

- 计算密度：dense 3D decode 中到高；sparse decode 因 metadata 和不规则访问常变成 memory/control-bound。
- 访存模式：dense grid 规则但容量巨大；sparse voxel 与沿轨迹 query 不连续；时序占用是状态访问。
- 并行性：voxel/cell/head 可并行；sparse 活跃集和 future rollout 存在负载不均。
- 数据复用：BEV 特征、地图先验、历史 occupancy 可复用；future horizon 增加状态驻留需求。
- 量化敏感度：特征/logits 可量化；occupied/free 边界、小障碍物、collision margin 和稀疏 metadata 不能当普通低比特张量处理。
- 瓶颈类型：dense 是 memory-capacity/bandwidth-bound；sparse 是 irregular-access/control-overhead-bound。
- 对硬件的核心需求：3D tiling、sparse metadata 支持、大容量 voxel cache、快速 collision query、planner 近端访问。

## 参考来源

- Wei et al., `SurroundOcc: Multi-camera 3D Occupancy Prediction for Autonomous Driving`, ICCV 2023 / arXiv:2303.09551。成熟度：已落地研究基线。
- Wang et al., `OpenOccupancy: A Large Scale Benchmark for Surrounding Semantic Occupancy Perception`, ICCV 2023 / arXiv:2303.03991。成熟度：已落地 benchmark。
- Zheng et al., `OccWorld: Learning a 3D Occupancy World Model for Autonomous Driving`, ECCV 2024 / arXiv:2311.16038。成熟度：研究阶段，4D occupancy 预测代表。
- Tian et al., `Occ3D: A Large-Scale 3D Occupancy Prediction Benchmark for Autonomous Driving`, NeurIPS 2023 / arXiv:2304.14365。成熟度：已落地 benchmark。
