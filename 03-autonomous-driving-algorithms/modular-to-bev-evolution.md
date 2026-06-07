# Modular to BEV Evolution

上级：[Autonomous Driving Algorithms](README.md)
相关：[BEV Perception](../02-vision-and-3d-perception/bev-perception.md), [BEV Workload](../06-chip-workload-analysis/bev-workload.md)

## 这页在回答什么问题

这页回答自动驾驶为什么从"感知-预测-规划-控制"的模块化 pipeline，走向以 BEV/Occupancy 为中心的统一空间表征。重点是模块接口如何限制信息流，以及 BEV 为什么把系统瓶颈从规整 CNN 推向 view transform 这类不规则访存——这是理解 03 整章 E2E 路线的起点。

## 为什么模块化会被取代：直觉与类比

模块化 pipeline 的直觉是**一条流水线上几个工位，每个工位把上一站的产物压成一张精简表格递给下一站**。感知工位看完原始图像，递给下游的不是图像，而是一张表：「3 个车、2 条车道线、1 个红灯、可通行区域如下」。预测工位拿这张表算出他车轨迹，再递给规划工位。

这个结构的好处显而易见——接口清晰、可调试、可验证，每个工位能单独测。但它有个致命的结构性缺陷：**表格太窄，且信息一旦在某站被丢掉，下游再也找不回来**。感知把原始传感器压成 box 和 lane 时，大量遮挡线索、不确定性、"那个东西说不清是什么但好像在动"的模糊上下文全被扔了。如果感知漏检了一个异形障碍（它不属于任何已知类别，填不进表格），规划工位手里的表上压根没有它，再聪明也躲不开——因为规划没机会去看更底层的 feature 自行判断。这就是模块化的根本病：**窄接口造成不可恢复的信息损失**。

BEV 的解法直觉是**把"逐站递表格"换成"所有工位围着同一块共享白板工作"**。感知不再把场景压成表格递走，而是把多相机信息投影、融合成一块 ego 中心俯视坐标系的场景白板（BEV feature），检测、建图、预测、规划都从这块白板上读自己要的东西。为什么这有效：白板保留了远比表格丰富的上下文，下游任务能从共享 feature 里恢复上游可能丢掉的信息，而且大家在同一个坐标系里对齐——规划本就在地面坐标系工作，BEV 正好把感知表示和规划坐标系合一（见 [BEV Perception](../02-vision-and-3d-perception/bev-perception.md)）。

常见误解：BEV 是一个感知 head。实际上 BEV 是中间表示革命——它改变了多摄像头融合方式、时序缓存策略、下游任务接口和整条芯片数据流。

## 从 BEV 到 Occupancy：接口继续变宽

BEV 仍是 2D 地平面表示，不完整表达高度、遮挡和 3D occupied/free/unknown 状态。Occupancy 把 BEV 推向 3D world state，更自然地描述异形障碍和可通行空间（见 [Occupancy Prediction](../02-vision-and-3d-perception/occupancy-prediction.md)）。workload 上这一步代价更大：BEV 的核心是 `camera feature → BEV grid`，Occupancy 进一步引入 `BEV/3D feature → voxel grid`，容量和带宽随 voxel 分辨率、语义通道、temporal horizon 放大，从 irregular-access-bound 叠加 capacity-bound。

整条演进线可以一句话概括：**接口从"窄而离散"（object/lane 表格）不断走向"宽而可微"（BEV feature → occupancy → 端到端可微图）**，模块边界逐步溶解。

## 一句话理解

从模块化到 BEV 的核心是从固定 object/lane 接口转向统一空间表征：它消除了窄接口造成的不可恢复信息损失，代价是把系统瓶颈从规整 CNN 推向 view transform、scatter/gather、BEV cache 和 3D tensor 容量。

## 演进与未来方向（判断）

以下为判断，锚定真实趋势。

主线判断很明确：**感知和规划之间的边界会持续溶解，终态是一张端到端可微的统一计算图**。模块化→BEV→Occupancy→planning-oriented E2E（见 [Planning-oriented E2E](planning-oriented-e2e.md)）是同一股力量在推——每一步都在拓宽接口、让上游表示对下游规划可微、减少模块边界的信息损失。再往前是 VLM reasoning 和 World Model rollout 也并入这张图（见 [VLM/VLA for AD](vlm-vla-for-ad.md)、[World Model for AD](world-model-for-ad.md)）。但有一条务实的边界：量产系统不会真的取消安全壳，可微大图外面始终套着 rule check、collision check、fallback planner——E2E 拓宽的是主链路的信息流，不是替代独立的安全机制。

对架构师，这条演进的硬件含义是一次**瓶颈性质的迁移**：模块化时代的主算力在规整 CNN/point encoder（compute-bound，NPU 擅长）；BEV/Occupancy 时代的签名瓶颈变成 view transform 的 scatter/gather 和 3D tensor 容量（irregular-access + capacity-bound，恰是规整 MAC 阵列的弱项）。所以"自动驾驶芯片"的设计重心，会随这条算法演进从"堆 TOPS"转向"高效支持不规则访存 + 大容量 activation + 共享 BEV cache 跨任务复用"——这正是 06 [BEV Workload](../06-chip-workload-analysis/bev-workload.md) 的出发点。在 archax 里，这条演进意味着整条 pipeline 不能再当成单一 compute-bound 工作点，必须显式拆出 view transform 这个 irregular-access 阶段单独建模。

## Workload Characterization

计算密度：传统模块中 CNN/point cloud encoder 可 compute-bound；BEV view transform 和 occupancy decode 常 memory/irregular-access-bound——这是演进带来的性质迁移。

访存模式：模块化 object list 轻但信息窄；BEV feature dense 且需 camera-to-BEV gather/scatter；temporal BEV/occupancy 需 state cache。

并行性：camera、feature level、BEV cell、task head 可并行；fusion 和 planning stage 有同步点。

数据复用：BEV feature 可被 detection/map/occupancy/planning 跨任务复用（共享白板是关键收益）；历史 BEV 可跨帧复用。

量化敏感度：encoder/head 可低比特；几何投影、depth、插值、occupancy boundary 需谨慎（几何误差直接放大成定位误差）。

瓶颈类型：BEV 阶段常 irregular-access + bandwidth-bound；Occupancy 进一步 capacity-bound——与模块化时代的 compute-bound 形成对照。

对硬件的核心需求：多路 camera DMA、projection table、scatter/gather、BEV SRAM/DRAM cache、3D tensor tiling，以及共享 BEV feature 跨任务复用的调度——详见 [BEV Workload](../06-chip-workload-analysis/bev-workload.md)。

## 参考来源

- Philion and Fidler, `Lift, Splat, Shoot`, ECCV 2020, arXiv:2008.05711。成熟度：已落地，几何投影 BEV 出处。
- Li et al., `BEVFormer`, ECCV 2022, arXiv:2203.17270。成熟度：已落地，query-based BEV 代表。
- Wei et al., `SurroundOcc`, ICCV 2023, arXiv:2303.09551。成熟度：已落地研究，BEV→occupancy。
- Hu et al., `Planning-oriented Autonomous Driving (UniAD)`, CVPR 2023, arXiv:2212.10156。成熟度：已落地研究，统一可微图代表。
