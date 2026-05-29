# Autonomous Driving Algorithms

上级：[Workload Wiki](../README.md)
相关：[BEV Perception](../02-vision-and-3d-perception/bev-perception.md), [Chip Workload Analysis](../06-chip-workload-analysis/README.md)

## 这章在回答什么问题

这一章回答自动驾驶算法路线如何从模块化 pipeline 演进到 BEV、E2E、VLM/VLA 和 World Model。这里不以“哪篇论文 SOTA”为主线，而是追踪表示方式和系统接口如何变化，以及这些变化如何改变 workload。

## 主线

```text
modular pipeline
  -> BEV / Occupancy representation
  -> planning-oriented E2E
  -> behavior cloning E2E
  -> VLM / VLA for AD
  -> world model for AD
  -> data closed loop and simulation
```

这条主线的本质是：越来越多的功能从人工接口和规则，迁移到统一 scene representation、temporal model、policy decoder 和 action-conditioned rollout。

## 页面列表

- [Modular to BEV Evolution](modular-to-bev-evolution.md)
- [Behavior Cloning E2E](behavior-cloning-e2e.md)
- [Planning-oriented E2E](planning-oriented-e2e.md)
- [VLM and VLA for Autonomous Driving](vlm-vla-for-ad.md)
- [World Model for Autonomous Driving](world-model-for-ad.md)
- [Data Closed Loop and Simulation](data-closed-loop-and-simulation.md)

## 到 06 的映射

| 路线 | 主要 workload | 06 对应入口 |
| --- | --- | --- |
| Modular / BEV | view transform、BEV encoder、multi-task heads | [BEV Workload](../06-chip-workload-analysis/bev-workload.md) |
| Planning-oriented E2E | shared scene encoder、query decoder、planning head | [E2E Workload](../06-chip-workload-analysis/e2e-workload.md) |
| Behavior Cloning E2E | video encoder、temporal model、trajectory decoder | [E2E Workload](../06-chip-workload-analysis/e2e-workload.md) |
| VLM/VLA for AD | visual tokens、language/route tokens、action decoder | [VLA Workload](../06-chip-workload-analysis/vla-workload.md) |
| World Model for AD | rollout、future state、risk/cost head | [World Model Workload](../06-chip-workload-analysis/world-model-workload.md) |
| Data closed loop | cloud training、simulation、scenario mining | [Cloud Inference and Simulation Chip](../06-chip-workload-analysis/cloud-inference-and-simulation-chip.md) |
