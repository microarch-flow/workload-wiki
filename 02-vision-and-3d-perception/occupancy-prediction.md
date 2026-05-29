# Occupancy Prediction

上级：[Vision and 3D Perception](README.md)
相关：[BEV Perception](bev-perception.md), [Occupancy Workload](../06-chip-workload-analysis/occupancy-workload.md), [World Model Workload](../06-chip-workload-analysis/world-model-workload.md)

## 这页在回答什么问题

Occupancy Prediction 回答三维空间中哪里被占据、哪里空闲、哪里未知，必要时还预测语义类别。它把感知从对象和像素推进到 3D world state，是自动驾驶和机器人走向 World Model 的关键表示。

## 计算结构

典型链路是：

```text
multi-view images / lidar / fused BEV
   ->
BEV or 3D representation
   ->
voxel / tri-plane / occupancy decoder
   ->
occupied / free / semantic occupancy
```

Occupancy 的输出可以是 dense 3D voxel，也可以是 sparse voxel、BEV+height、tri-plane 或 implicit representation。Camera-only occupancy 依赖深度和多视角几何推断，multi-modal occupancy 利用 LiDAR 几何补强。

典型量级上，若用 `200 x 200 x 16` voxel grid，就已经有 64 万个 cell；如果每个 cell 输出 18 个语义 logits，单帧 logits 就超过 1100 万个值。若再引入 4-8 帧 temporal occupancy 或 future occupancy，容量和带宽会按 horizon 线性放大，快速变成 memory-capacity-bound。

## 为什么它重要

检测只能表达离散对象，难覆盖异形障碍物、可通行空间和未知区域。Occupancy 直接描述空间状态，更接近 planning 和 collision checking 所需的信息。

Occupancy 的表示选择会直接改变 workload。Dense voxel 访问规则但容量爆炸，适合固定 grid 和规整 decoder；sparse voxel 只计算非空或高置信区域，但需要 metadata、hash/rulebook 和负载均衡；tri-plane 把 3D 分解到多个 2D plane，降低容量但需要跨平面采样；implicit/NeRF-like representation 通过 query 采样表达连续场，但推理时 query 数和采样点会变成不规则访问。

在 World Model 中，future occupancy prediction 是最自然的落地形式之一：不必生成逼真视频，只要预测未来空间占据和风险状态即可。OccWorld 这类路线已经把 occupancy token 当成 world state，预测未来 occupancy 和 ego trajectory；这说明 occupancy 是 02 感知到 05/06 World Model workload 的关键桥梁。

常见误解：Occupancy 是 3D 语义分割。实际上，它还要表达 free/unknown/occupied，且目标通常是服务规划和世界状态，而不只是给每个 voxel 分类。

## 一句话理解

Occupancy 把场景表示从对象级和图像级推进到 3D 空间状态；它的 workload 核心是 3D/4D 张量容量、稀疏表示和 voxel decode。

## Workload Characterization

- 计算密度：dense 3D decoder 和 voxel attention 计算重；sparse occupancy 降低计算但引入 metadata 和索引开销。
- 访存模式：dense voxel 规则但容量大；sparse voxel 需要 metadata/rulebook；tri-plane 需要跨 plane gather；implicit query 不规则；temporal occupancy 需要状态缓存。
- 并行性：voxel/cell、semantic channel、camera、time horizon 可并行；稀疏分布会造成负载不均。
- 数据复用：BEV feature、depth feature、temporal state 可复用；3D decoder 中间 feature 占用大。
- 量化敏感度：occupancy logits 可低比特；几何坐标、depth、free/unknown boundary 和 sparse metadata 需谨慎。
- 瓶颈类型：dense occupancy 常 memory-capacity-bound；sparse occupancy 常 irregular-access-bound；future occupancy 可能 rollout-latency-bound。
- 对硬件的核心需求：大容量 activation buffer、3D/4D tensor tiling、sparse metadata/rulebook、BEV-to-voxel decode、tri-plane sampling、低延迟 temporal cache。

## 参考来源

- Wei et al., `SurroundOcc: Multi-Camera 3D Occupancy Prediction for Autonomous Driving`, ICCV 2023, arXiv:2303.09551。
- Huang et al., `Tri-Perspective View for Vision-Based 3D Semantic Occupancy Prediction`, CVPR 2023, arXiv:2302.07817。
- Cao and de Charette, `MonoScene: Monocular 3D Semantic Scene Completion`, CVPR 2022, arXiv:2112.00726。
- Tian et al., `Occ3D: A Large-Scale 3D Occupancy Prediction Benchmark for Autonomous Driving`, NeurIPS 2023 Datasets and Benchmarks, arXiv:2304.14365。
- Wang et al., `OpenOccupancy: A Large Scale Benchmark for Surrounding Semantic Occupancy Perception`, ICCV 2023, arXiv:2303.03991。
- Zheng et al., `OccWorld: Learning a 3D Occupancy World Model for Autonomous Driving`, ECCV 2024, arXiv:2311.16038。
