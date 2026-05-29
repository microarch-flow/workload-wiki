# BEV Workload

上级：[Chip Workload Analysis](README.md)
相关：[BEV Perception](../02-vision-and-3d-perception/bev-perception.md), [E2E Workload](e2e-workload.md)

## 这页在回答什么问题

这页分析 BEV workload 的芯片压力。BEV 的难点不是 backbone FLOPs，而是多相机 feature 到 ego-centric BEV grid 的投影、采样、散射、融合和时序缓存。这些 stage 会把规则 CNN/Transformer workload 变成 irregular memory workload。

## Stage 拆解

| Stage | 输入输出 | 主导成本 |
| --- | --- | --- |
| multi-camera encoder | `Ncam x image -> feature pyramid` | CNN/ViT compute + activation bandwidth |
| view transform | image feature -> BEV cell | gather/scatter、depth bins、index table |
| BEV fusion | multi-camera/multi-scale -> BEV feature | memory bandwidth、NoC 分发 |
| temporal BEV | history BEV alignment/cache | state capacity、ego-motion warp |
| task heads | detection/map/occupancy/planning | shared feature reuse |

典型参数包括 `camera count = 6-8`、输入分辨率、feature scale、depth bins、BEV grid，例如 `200 x 200` 或更大，以及 history frame 数。参数增加时，view transform 的索引访问和 BEV cache 容量通常比单纯 backbone FLOPs 更早成为瓶颈。

## 为什么 BEV 对硬件不友好

标准 CNN/ViT encoder 的访问较规则，可以 tile。BEV view transform 要根据相机内外参、depth 或 learned query，把 image feature 映射到 BEV cell。这个过程可能包含 bilinear sampling、scatter add、deformable attention 或 sparse gather，访问位置由几何表或 query 决定，DRAM row locality 较差。

如果 BEV feature 被多个 head 复用，收益很大；但 producer-consumer buffer 必须设计好，否则 detection、map、occupancy、planning 反复从 DRAM 读同一 BEV feature，会把共享表征优势抵消。

## 硬件连接

- RAM：BEV cache 是长生命周期状态；view transform 索引访问破坏 row locality。
- DMA：需要 2D/3D stride、scatter-gather、descriptor chaining、camera feature 到 BEV tile 的预取。
- NOC：多 camera feature 汇聚到 BEV tile，需要 multicast/reduction 和拥塞控制。
- CIM：encoder 中的 Conv/GEMM 可用 CIM；view transform、scatter/gather、ego-motion warp 不适合作为 CIM 主收益点。
- PCIE/host：相机标定、projection table 应在端侧预处理并常驻，避免逐帧 host 往返。

## archax 建模

- Resource：camera input bandwidth、encoder TOPS、BEV SRAM/DRAM cache、DMA scatter-gather engine、NoC bandwidth。
- Topology：camera encoder tile 到 BEV fusion tile 的连接，BEV cache 所在 memory level。
- Interaction：`camera feature -> projection/gather -> BEV update -> multi-task heads`。
- Capability：indexed load/store、scatter add、deformable attention、ego-motion alignment、INT8 encoder、FP16/FP32 geometry index。

## Workload Characterization

- 计算密度：encoder 偏 compute-bound；view transform 偏 irregular-access/memory-bound；BEV heads 中等。
- 访存模式：multi-camera feature 连续，projection/gather/scatter 不连续，temporal BEV 是状态访问。
- 并行性：camera、feature level、BEV cell、task head 可并行；fusion 和 temporal update 有同步点。
- 数据复用：BEV feature 可被 detection/map/occupancy/planning 复用；projection table 和历史 BEV 可跨帧复用。
- 量化敏感度：encoder/head 可 INT8；geometry、depth、插值和边界对齐需保守。
- 瓶颈类型：第一瓶颈常是 irregular-access；第二瓶颈是 BEV cache bandwidth/capacity。
- 硬件核心需求：scatter-gather DMA、BEV state cache、NoC 多源汇聚、低延迟多 camera pipeline。

## 参考来源

- Philion and Fidler, `Lift, Splat, Shoot`, ECCV 2020 / arXiv:2008.05711。
- Li et al., `BEVFormer`, ECCV 2022 / arXiv:2203.17270。
- Liu et al., `BEVFusion`, ICRA 2023 / arXiv:2205.13542。
- `/mnt/e/workload-wiki-old/06_芯片架构Workload分析/BEV_Workload.md`。
