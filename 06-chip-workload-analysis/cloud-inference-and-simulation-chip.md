# Cloud Inference and Simulation Chip

上级：[Chip Workload Analysis](README.md)
相关：[World Model Workload](world-model-workload.md), [Edge-cloud Collaborative World Model](../05-world-model-and-generative/edge-cloud-collaborative-world-model.md)

## 这页在回答什么问题

这页分析云端推理与仿真芯片需求。云端 workload 与端侧不同：它可以用大 batch、异步调度和高功耗换吞吐，但要处理更大的模型、更长上下文、更大数据湖和大规模仿真。

## 云端 workload 类型

| 类型 | 例子 | 主导成本 |
| --- | --- | --- |
| large model inference | VLM/VLA、LLM、World Model prefill | GEMM compute、HBM bandwidth |
| data mining | fleet log、robot log、scenario search | storage IO、batch inference |
| simulation | nuPlan/NAVSIM/CARLA/Waymo World Model | 多进程、多模型、多场景并行 |
| generative world model | video/occupancy/latent generation | diffusion/transformer + activation capacity |
| training/fine-tuning | VLA imitation、world model training | 显存、通信、数据吞吐 |

云端的关键指标不是单次 latency，而是 cost per scenario、tokens/sec、frames/sec、rollouts/sec 和 training throughput。

## 与端侧的差异

云端可以通过 batch 提高 GEMM 利用率，因此 Transformer prefill、VLM encoder、Diffusion denoise 更容易吃满计算阵列。端侧最困难的 batch=1 decode，在云端可以通过多请求合批缓解。相反，云端更容易被显存容量、HBM bandwidth、storage IO、跨卡通信和调度效率限制。

## 硬件连接

- RAM/HBM：大模型权重、activation、KV cache、video latent、simulation state 要求高容量和高带宽。
- DMA：大批量 tensor、日志数据、视频/点云解码需要高吞吐搬运。
- NOC/互连：多卡训练和仿真需要 all-reduce、parameter sharding、pipeline 并行。
- CIM：规则 GEMM/Conv 有潜力，但云端大模型生态更依赖通用矩阵引擎和软件栈。
- PCIE/host：数据加载、storage、NIC、GPU/NPU 间边界是云端总吞吐的重要部分。

## archax 建模

- Resource：HBM capacity/bandwidth、compute throughput、interconnect、host IO、storage throughput。
- Topology：multi-device fabric、host-device-storage 网络、simulation workers。
- Interaction：batch inference、model parallel、data pipeline、rollout simulation loop、artifact writeback。
- Capability：large GEMM、attention、diffusion loop、multi-instance scheduling、mixed precision、collective communication。

## Workload Characterization

- 计算密度：大 batch dense compute 高；生成模型多步循环放大总计算。
- 访存模式：大块连续 tensor 为主，但日志/视频/点云数据加载会形成系统 IO 压力。
- 并行性：batch、scenario、sample、model version、rollout 可高度并行；训练有通信同步。
- 数据复用：模型权重在 batch 内复用；world model condition encoding 可跨 sample 复用。
- 量化敏感度：云端推理可 INT8/FP8；训练常 BF16/FP8；安全评估指标需数值一致。
- 瓶颈类型：训练是 memory capacity + communication；仿真是 compute + IO + scheduling；批量推理是 HBM bandwidth + GEMM utilization。
- 硬件核心需求：HBM、大规模 GEMM、集群互连、数据管线、仿真调度、多租户隔离。

## 参考来源

- Caesar et al., `nuPlan`, arXiv:2106.11810。
- Dauner et al., `NAVSIM`, NeurIPS 2024 / arXiv:2406.15349。
- NVIDIA, `Cosmos World Foundation Models`, 2025，成熟度：产业平台，查证日期：2026-05-29。
- `/mnt/e/workload-wiki-old/06_芯片架构Workload分析/云端推理与仿真芯片需求.md`。
