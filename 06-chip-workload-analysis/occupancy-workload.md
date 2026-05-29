# Occupancy Workload

上级：[Chip Workload Analysis](README.md)
相关：[Occupancy Prediction](../02-vision-and-3d-perception/occupancy-prediction.md), [BEV Workload](bev-workload.md)

## 这页在回答什么问题

这页分析 Occupancy workload。Occupancy 把 scene representation 从 2D BEV 推到 3D occupied/free/unknown 或 semantic occupancy，核心压力从平面 feature 变成 3D/4D tensor 的容量、稀疏访问和 future horizon。

## Stage 拆解

| Stage | 输入输出 | 主导成本 |
| --- | --- | --- |
| sensor/BEV encoder | camera/LiDAR -> BEV/voxel feature | encoder compute、view transform |
| 3D lifting | BEV/image feature -> voxel/tri-plane | capacity、gather/scatter |
| occupancy decoder | voxel feature -> occupancy logits | 3D conv/sparse conv/transformer |
| temporal prediction | history -> future occupancy | state capacity、horizon |
| query/check | trajectory -> occupancy collision | random query、latency |

一个 `200 x 200 x 16` grid 已经有 64 万 cell；如果每个 cell 有 semantic channel、history 和 future horizon，activation 和 logits 很快超过片上缓存。稀疏表示可以减少计算，但会引入 metadata、index、负载不均和 gather/scatter。

## dense 与 sparse 的取舍

dense occupancy 接口简单，适合规则 3D tensor tiling，但容量压力大。sparse occupancy 更贴合真实世界占用分布，但对 deterministic NPU 更难：活跃 voxel 分布随场景变化，metadata 解码、indice pair、rulebook 和负载均衡会造成 control overhead。

因此架构建模不能只看 dense 3D FLOPs，也要记录 occupancy 的有效稀疏率、活跃 voxel 分布、future horizon 和 query 模式。

## 硬件连接

- RAM：dense grid capacity 是第一问题；sparse metadata 会造成 row locality 下降。
- DMA：需要 3D tile 搬运、sparse block 搬运、metadata 与 feature 协同搬运。
- NOC：3D tile 间 halo exchange、sparse block 分发、query result 汇聚会产生通信。
- CIM：dense 3D conv 局部适合 CIM；sparse metadata 和动态 query 不适合。
- PCIE/host：collision query 和 planner 如果在 CPU，occupancy logits 往返会造成同步瓶颈。

## archax 建模

- Resource：voxel cache 容量、3D conv TOPS、sparse metadata buffer、DRAM bandwidth。
- Topology：3D grid tile 到 NPU tile 的映射，planner/query 与 occupancy memory 的连接。
- Interaction：`BEV/voxel feature -> occupancy decode -> future cache -> trajectory query`。
- Capability：3D conv、sparse conv、indexed query、metadata decode、multi-resolution tile、INT8 logits + high precision geometry。

## Workload Characterization

- 计算密度：dense 3D decode 中等到高；sparse decode 可能因 metadata 和不规则访问变成 memory/control-bound。
- 访存模式：dense grid 规则但容量大；sparse voxel 和 trajectory query 不连续。
- 并行性：voxel/cell/head 可并行；sparse active set 和 future rollout 有负载不均。
- 数据复用：BEV feature、map prior、history occupancy 可复用；future horizon 增加状态驻留需求。
- 量化敏感度：feature/logits 可量化；occupied/free 边界、小障碍、collision margin 和 metadata 不等同于普通低比特 tensor。
- 瓶颈类型：dense 是 memory-capacity/bandwidth-bound；sparse 是 irregular-access/control-overhead-bound。
- 硬件核心需求：3D tiling、sparse metadata 支持、voxel cache、快速 collision query、planner 近端访问。

## 参考来源

- Wei et al., `SurroundOcc`, ICCV 2023 / arXiv:2303.09551。
- Wang et al., `OpenOccupancy`, ICCV 2023 / arXiv:2303.03991。
- Zheng et al., `OccWorld`, arXiv:2311.16038。
- `/mnt/e/workload-wiki-old/06_芯片架构Workload分析/Occupancy_Workload.md`。
