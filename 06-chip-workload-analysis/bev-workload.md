# BEV Workload

上级：[Chip Workload Analysis](README.md)
相关：[BEV Perception](../02-vision-and-3d-perception/bev-perception.md), [E2E Workload](e2e-workload.md)

## 这页在回答什么问题

这页分析 BEV 感知的芯片压力。BEV 的难点不在 backbone 的 FLOPs，而在把多相机透视特征投影到 ego-centric BEV 栅格的那一步——它会把原本规则、可 tile 的 CNN/ViT workload，变成一个 row locality 很差的 irregular memory workload。理解这一步，才能解释为什么 BEV 芯片的瓶颈常常不是算力，而是 scatter-gather DMA 和 BEV cache 带宽。

## view transform 为什么是 BEV 的真正瓶颈

标准 CNN/ViT encoder 的访问是规则的，可以干净地 tile。但 view transform 要根据相机内外参，把每个 image 像素特征送到它在 BEV 平面上对应的栅格——而"对应关系"由几何决定，是数据相关的、不连续的。

两条主流路线对硬件的压力点不同。lift-splat 类（如 Lift-Splat-Shoot、BEVDet）为每个像素预测一组 depth bin，把特征沿视线"提升"成伪点云，再"散射"（scatter-add）到 BEV 栅格——这是典型的 scatter 操作，写地址由几何表决定，多个像素可能写同一个 BEV cell 产生写冲突，DRAM row locality 几乎被打散。query 类（如 BEVFormer）反过来，让每个 BEV query 通过 deformable attention 去多个相机、多个尺度上 gather 采样点——这是 gather 操作，读地址由 query 的可学习偏移决定，同样不规则。无论哪条路线，访问位置都不是编译期可知的连续块，而是由几何表或 query 动态决定的离散地址。

具体规模感：典型环视配置是 6–8 路相机，BEV 栅格常取 `200 × 200`（nuScenes 量级，约 4 万 cell）甚至更大，depth bin 数十个，再叠加多尺度特征金字塔。当这些参数增大时，**view transform 的索引访问量和 BEV cache 容量，通常比 backbone 的 FLOPs 更早成为瓶颈**。

## 共享 BEV 特征：收益与陷阱并存

BEV 表示的最大价值是一份特征被检测、建图、占用、规划多个 head 复用，这正是 archax"数据搬运优先"原则最该兑现收益的地方。但收益不是自动的：如果 producer-consumer buffer 没设计好，多个 head 各自从 DRAM 重新读同一份 BEV 特征，共享表征的优势会被重复搬运抵消。所以 BEV 芯片设计的一个关键问题是——这份 BEV 特征能否常驻 SRAM，让所有下游 head 就近消费。

时序 BEV（temporal BEV）又叠加一层状态压力：要把历史帧的 BEV 特征按 ego-motion 做 warp 对齐后融合，于是 BEV cache 变成一个长生命周期、需要按运动更新的状态，容量和带宽都随 history window 增长。

## 可建模参数

`camera count`（6–8）和 `image resolution` 决定 encoder 算力与输入带宽；`feature scale`（金字塔层数）决定多尺度搬运量；`depth bins` 决定 lift 的伪点云规模；`BEV grid`（如 200×200）决定 scatter 目标空间和 BEV cache 容量；`history window` 决定时序 cache 的状态驻留量；`sampling pattern`（lift-splat 的 scatter vs query 的 gather）决定不规则访问的具体形态。这些才是架构探索真正要扫描的变量，而非笼统的"BEV 模型大小"。

## 硬件连接

RAM：BEV cache 是长生命周期状态，最好能常驻 SRAM 供多 head 复用；view transform 的索引访问会破坏 row locality，是 RAM wiki row-buffer 分析里典型的 row-miss 高发场景。DMA：必须支持 2D/3D stride、scatter-gather、descriptor chaining，以及相机特征到 BEV tile 的预取（见 DMA wiki）——这是 BEV 能否高效的决定性能力。NOC：多相机特征汇聚到 BEV tile 需要 multicast/reduction 和拥塞控制，否则环视融合阶段会出现阶段性带宽尖峰（见 NOC wiki）。CIM：encoder 里的 Conv/GEMM 适合 CIM，但 view transform 的 scatter/gather 和 ego-motion warp 不是 CIM 的收益点。PCIE/host：相机标定与 projection 索引表应在端侧预处理并常驻，绝不能逐帧回 host 重算。

## archax 建模

Resource：相机输入带宽、encoder TOPS、BEV SRAM/DRAM cache、DMA scatter-gather engine、NoC 带宽。Topology：相机 encoder tile 到 BEV fusion tile 的连接，以及 BEV cache 落在哪一层 memory。Interaction：`相机特征 → projection/gather/scatter → BEV update → 多任务 heads` 的链路，时序分支是 `history BEV → ego-motion warp → fuse`。Capability：indexed load/store、scatter-add、deformable attention、ego-motion alignment、INT8 encoder 配合 FP16/FP32 的几何索引计算。这里 archax 探索的关键不是 TOPS，而是 scatter-gather DMA 能力和 BEV cache residency 这两个开关。

## 一句话理解

BEV 的 workload 重心从 backbone 算力转移到了"把透视特征不规则地搬到 BEV 栅格"这一步——它是 irregular-access-bound 的，芯片成败取决于 scatter-gather DMA 与 BEV cache，而非峰值 TOPS。

## Workload Characterization

- 计算密度：encoder 偏 compute-bound；view transform 偏 irregular-access/memory-bound；BEV head 中等。
- 访存模式：多相机特征连续可 tile，但 projection/gather/scatter 不连续、row locality 差，时序 BEV 是状态访问。
- 并行性：相机、特征层级、BEV cell、task head 可并行；融合与时序更新是同步点。
- 数据复用：BEV 特征可被检测/建图/占用/规划复用（核心收益）；projection 表和历史 BEV 可跨帧复用，但需 SRAM 常驻才兑现。
- 量化敏感度：encoder/head 可 INT8；depth、坐标、插值与边界对齐这些几何量需保守精度，否则空间对齐漂移。
- 瓶颈类型：第一瓶颈通常是 irregular-access；第二瓶颈是 BEV cache 的 bandwidth/capacity；compute 一般不是第一瓶颈。
- 对硬件的核心需求：scatter-gather DMA、可常驻的 BEV state cache、NOC 多源汇聚与 QoS、低延迟多相机 pipeline。

## 参考来源

- Philion and Fidler, `Lift, Splat, Shoot`, ECCV 2020 / arXiv:2008.05711。成熟度：已落地，lift-splat 范式代表。
- Li et al., `BEVFormer: Learning Bird's-Eye-View Representation from Multi-Camera Images via Spatiotemporal Transformers`, ECCV 2022 / arXiv:2203.17270。成熟度：已落地，query/deformable-attention 范式代表。
- Liu et al., `BEVFusion: Multi-Task Multi-Sensor Fusion with Unified Bird's-Eye View Representation`, ICRA 2023 / arXiv:2205.13542。成熟度：已落地，多传感器 BEV 融合代表。
- Huang et al., `BEVDet: High-performance Multi-camera 3D Object Detection in Bird-Eye-View`, arXiv:2112.11790。成熟度：已落地工程实现。
