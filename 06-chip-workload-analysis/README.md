# Chip Workload Analysis

上级：[Workload Wiki](../README.md)
相关：[Workload Analysis Methodology](workload-analysis-methodology.md), [Workload Characterization Axes](workload-characterization-axes.md)

## 这章在回答什么问题

这一章是整份 wiki 的重心：把 CNN、Transformer、BEV、Occupancy、E2E 自动驾驶、VLA、World Model 等算法，翻译成芯片架构探索可以建模的 workload。前面 01-05 解释算法为什么这样设计；本章回答这些设计如何落到 RAM、DMA、NOC、CIM、PCIE 和 archax 的 Resource / Topology / Interaction / Capability。

## 本章方法

每个 workload 都按同一流程展开：

```text
algorithm dataflow
  -> stage decomposition
  -> dominant cost
  -> workload characterization
  -> hardware wiki connection
  -> archax modeling
```

这里的重点不是列算子，而是判断瓶颈。标准 Conv 和 Transformer prefill 可能 FLOPs 很高但适合 NPU；BEV gather/scatter、KV cache decode、3D occupancy、action token decode 可能 FLOPs 不最高，却更容易决定 p99 latency、带宽和容量。

## 页面列表

- [Workload Analysis Methodology](workload-analysis-methodology.md)
- [Workload Characterization Axes](workload-characterization-axes.md)
- [CNN Workload](cnn-workload.md)
- [Transformer Workload](transformer-workload.md)
- [BEV Workload](bev-workload.md)
- [Occupancy Workload](occupancy-workload.md)
- [E2E Workload](e2e-workload.md)
- [VLA Workload](vla-workload.md)
- [World Model Workload](world-model-workload.md)
- [Edge Inference Chip Requirements](edge-inference-chip-requirements.md)
- [Cloud Inference and Simulation Chip](cloud-inference-and-simulation-chip.md)
- [AD and Robotics Chip Architecture Summary](ad-robotics-chip-architecture-summary.md)

## Workload 对硬件 wiki 的连接

| Workload 现象 | RAM | DMA | NOC | CIM | PCIE / host |
| --- | --- | --- | --- | --- | --- |
| 大块 Conv/GEMM | row locality、bank parallelism | burst、double buffer | tile 分发 | 适配度高 | 低 |
| KV cache / state cache | 容量与带宽 | cache update | shared state path | 适配度低 | 低到中 |
| BEV gather/scatter | row miss 风险 | scatter-gather descriptor | 跨 tile feature 分发 | 适配度低 | 中 |
| 3D occupancy | 容量压力 | 3D tile 搬运 | sparse/dense 分布 | 局部 dense 可用 | 中 |
| VLA / LLM decode | 权重与 KV 带宽 | 小块连续搬运 | decode state 分发 | GEMM 局部可用 | 中 |
| 云端仿真 | 数据湖吞吐 | 大批量搬运 | 多卡通信 | 取决于模型 | 高 |

## archax 建模入口

本章显式使用四个抽象：

- Resource：compute array、SRAM、DRAM/HBM、DMA engine、NoC link、host interface、CIM array。
- Topology：tile mesh、memory hierarchy、producer-consumer stage、host-device boundary、edge-cloud boundary。
- Interaction：tensor transfer、cache update、rollout loop、decode loop、CPU/NPU synchronization。
- Capability：Conv/GEMM/Attention、scatter-gather、sparse metadata、quantization、stateful decode、online softmax。

每篇 workload 文档都要把算法参数写成可建模参数，例如 `sequence length`、`BEV grid`、`voxel resolution`、`camera count`、`candidate count`、`rollout horizon`、`action chunk length`。这些参数才是架构探索时真正扫描的变量。
