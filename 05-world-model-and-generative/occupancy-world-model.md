# Occupancy World Model

上级：[World Model and Generative Intelligence](README.md)
相关：[Occupancy Prediction](../02-vision-and-3d-perception/occupancy-prediction.md), [Occupancy Workload](../06-chip-workload-analysis/occupancy-workload.md)

## 这页在回答什么问题

这页回答 Occupancy World Model 为什么是自动驾驶和机器人安全评估的重要形式。它预测未来 3D 空间的 occupied/free/unknown 状态，比 2D BEV 更直接表达遮挡、高度和可通行空间。

## 基本结构

```text
history BEV / voxel / sensor feature
   ->
3D spatial-temporal dynamics
   ->
future occupancy grid
   ->
collision check / planning cost / simulation
```

Occupancy World Model 的输出可以是 dense voxel、sparse voxel、tri-plane、implicit field 或 hybrid representation。自动驾驶常关心 ego 周围几十到上百米范围，机器人常关心近场物体、桌面、手爪周边和接触区域。

## 价值和代价

价值是安全清晰：planner 可以直接查询某个 future pose 是否占用。代价是状态体积大。即使 `200 x 200 x 16` 的 occupancy grid，也已经是 64 万 cell；再乘以 semantic channel、future horizon 和 batch，访存压力会迅速变成主问题。

常见误解：occupancy 越 dense 越安全。实际上，dense grid 提供简单接口，但会带来巨大 memory footprint；工程上经常需要 sparse、multi-resolution 或 region-of-interest 表示。

## 一句话理解

Occupancy World Model 用 3D 空间占用预测未来风险；它最接近安全约束，但也是 memory capacity 和 bandwidth 压力最大的 world model 表示之一。

## Workload Characterization

- 计算密度：3D conv、sparse conv、voxel transformer 或 implicit decoder 构成主计算；dense 3D head 的 FLOPs 和 activation 都大。
- 访存模式：voxel grid 容量大，sparse 表示又带来 irregular access；future horizon 会线性放大中间状态。
- 并行性：voxel/cell 级并行强；sparse rulebook、neighbor query 和 temporal update 有同步与负载不均问题。
- 数据复用：history occupancy、map prior、BEV feature 可跨 future step 和 candidate 复用。
- 量化敏感度：大部分 feature 可量化；occupied/free 边界、小障碍和 collision margin 对误差敏感。
- 瓶颈类型：dense occupancy 常 memory-capacity/bandwidth-bound；sparse occupancy 常 irregular-access-bound。
- 对硬件的核心需求：3D tensor tiling、sparse metadata 处理、voxel cache、future horizon 并发、快速 collision query。

## 参考来源

- Wei et al., `SurroundOcc: Multi-Camera 3D Occupancy Prediction for Autonomous Driving`, ICCV 2023 / arXiv:2303.09551，https://arxiv.org/abs/2303.09551，成熟度：occupancy 基础研究，查证日期：2026-05-29。
- Zheng et al., `OccWorld: Learning a 3D Occupancy World Model for Autonomous Driving`, arXiv:2311.16038，https://arxiv.org/abs/2311.16038，成熟度：occupancy world model 代表，查证日期：2026-05-29。
- `OpenOccupancy: A Large Scale Benchmark for Surrounding Semantic Occupancy Perception`, ICCV 2023 / arXiv:2303.03991，https://arxiv.org/abs/2303.03991，成熟度：常用数据/评测资源，查证日期：2026-05-29。
- `GEM: A Generalizable Ego-Vision Multimodal World Model for Fine-Grained Ego-Motion, Object Dynamics, and Scene Composition Control`, arXiv:2605.07326，https://arxiv.org/abs/2605.07326，成熟度：2026 前沿多模态 world model，查证日期：2026-05-29。

## 旧版素材

- `/mnt/e/workload-wiki-old/05_World_Model与生成式智能/Occupancy_World_Model.md`
