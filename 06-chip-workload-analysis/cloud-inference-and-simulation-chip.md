# Cloud Inference and Simulation Chip

上级：[Chip Workload Analysis](README.md)
相关：[World Model Workload](world-model-workload.md), [Edge-Cloud Collaborative World Model](../05-world-model-and-generative/edge-cloud-collaborative-world-model.md)

## 这页在回答什么问题

这页分析云端推理与仿真芯片需求。云端 workload 与端侧的约束几乎相反：它可以用大 batch、异步调度和高功耗换吞吐，但要处理更大的模型、更长的上下文、更大的数据湖和大规模仿真。云端的成败标准不是单次延迟，而是单位成本下的吞吐——cost per scenario、tokens/sec、frames/sec、rollouts/sec、training throughput。

## 大 batch 如何反转端侧的瓶颈

端侧最难的 batch=1 decode 是 memory-bandwidth-bound 的（见 [Transformer Workload](transformer-workload.md)）：每生成一个 token 都要把全部权重读一遍，算力喂不满。云端把多个请求合批（continuous batching），让同一次权重读取服务 batch 个 token，arithmetic intensity 随 batch 线性上升，于是 decode 重新变得 compute-friendly，GEMM 利用率被拉起来。Transformer prefill、VLM encoder、Diffusion 去噪在大 batch 下都更容易吃满计算阵列。

但合批不是没有代价：它把瓶颈从带宽转移到了别处。云端更容易被 HBM 容量（权重 + 所有并发请求的 KV cache）、HBM 带宽、storage IO（数据湖加载）、跨卡通信（tensor/pipeline 并行的 all-reduce）和调度效率限制。所以云端芯片的核心资源是 HBM 容量/带宽和互连，而不是单纯的 MAC 峰值。

## 云端 workload 的几种形态

大模型推理服务（VLM/VLA、LLM、World Model 的 prefill）瓶颈在 GEMM 算力与 HBM 带宽，靠合批提升利用率。数据挖掘（车队日志、机器人日志、长尾场景搜索）是 storage IO + 批量推理主导，吞吐看的是能多快扫完数据湖。仿真（nuPlan、NAVSIM、CARLA，以及 World Model 驱动的闭环仿真）是多进程、多模型、多场景并行，瓶颈在 compute + IO + 调度。生成式 World Model（视频/占用/latent 生成）是 diffusion/transformer 大模型叠加巨大的 activation 容量。训练/微调（VLA 模仿学习、World Model 训练）则是显存容量 + 跨卡通信 + 数据吞吐的经典训练三角。这几类形态对硬件的侧重不同，云端芯片选型要先确定主力形态。

## 与端侧的分工：端云协同的 workload 边界

云端和端侧不是同一 workload 的大小版本，而是分工。重的生成与仿真（World Model 数据引擎、长尾场景生成、大模型训练）留在云端；轻的实时推理（感知、规划、控制、端侧 VLA）放端侧。两者通过数据闭环连接：端侧采集 → 云端挖掘/生成/训练 → 模型下发端侧（见 [Edge-Cloud Collaborative World Model](../05-world-model-and-generative/edge-cloud-collaborative-world-model.md)）。架构上这意味着云端芯片要优化的是"离线吞吐与成本"，端侧芯片要优化的是"在线延迟与功耗"，把云端优化直接迁到端侧（或反之）通常是错的。

## 可建模参数

`model size` 与 `context length` 决定权重与 KV cache 的 HBM 占用；`batch / concurrency` 决定 GEMM 利用率与调度复杂度；`scenario / sample count` 决定仿真并行规模；`denoise steps × frames` 决定生成式 workload 的总计算；`device count` 与 `parallelism strategy`（data/tensor/pipeline）决定互连压力；`dataset size` 决定 storage IO。

## 硬件连接

RAM/HBM：大模型权重、activation、KV cache、video latent、simulation state 都要求高容量和高带宽，HBM 是云端的核心资源（见 RAM wiki）。DMA：大批量 tensor、日志数据、视频/点云解码需要高吞吐搬运（见 DMA wiki）。NOC/互连：多卡训练和仿真需要 all-reduce、parameter sharding、pipeline 并行的高带宽低延迟互连（见 NOC wiki，并外延到片间/卡间网络）。CIM：规则 GEMM/Conv 有潜力，但云端大模型生态更依赖通用矩阵引擎和成熟软件栈，CIM 的落地门槛更高。PCIE/host：数据加载、storage、NIC、加速器间边界是云端总吞吐的重要组成（见 PCIE wiki）。

## archax 建模

Resource：HBM 容量/带宽、compute throughput、interconnect 带宽、host IO、storage throughput。Topology：multi-device fabric、host-device-storage 网络、simulation worker 拓扑。Interaction：batch inference、model parallel 通信、data pipeline、rollout simulation loop、artifact writeback。Capability：大 GEMM、attention、diffusion 循环、multi-instance 调度、混合精度、collective communication。archax 在云端的目标函数是吞吐/成本与可扩展性，扫描变量是 batch、并行策略、HBM 容量和互连带宽。

## 一句话理解

云端芯片用大 batch 把端侧的带宽瓶颈反转成算力可吃满，但代价是瓶颈转移到 HBM 容量/带宽、互连和数据 IO；它优化的是吞吐与成本，与端侧优化延迟与功耗形成分工。

## Workload Characterization

- 计算密度：大 batch dense compute 高；生成模型的多步循环放大总计算。
- 访存模式：大块连续 tensor 为主，但日志/视频/点云的数据加载形成系统级 IO 压力。
- 并行性：batch、scenario、sample、model version、rollout 高度并行；训练有通信同步。
- 数据复用：模型权重在 batch 内复用；World Model 的 condition encoding 可跨 sample 复用。
- 量化敏感度：推理可 INT8/FP8；训练常 BF16/FP8；安全评估指标需数值一致性。
- 瓶颈类型：训练是 capacity + communication；仿真是 compute + IO + scheduling；批量推理是 HBM bandwidth + GEMM utilization。
- 对硬件的核心需求：大容量高带宽 HBM、大规模 GEMM、高带宽集群互连、数据管线、仿真调度、多租户隔离。

## 参考来源

- Pope et al., `Efficiently Scaling Transformer Inference`, MLSys 2023 / arXiv:2211.05102。成熟度：已落地，大 batch 推理与并行分析。
- Kwon et al., `Efficient Memory Management for Large Language Model Serving with PagedAttention (vLLM)`, SOSP 2023 / arXiv:2309.06180。成熟度：已落地服务系统。
- Dauner et al., `NAVSIM: Data-Driven Non-Reactive Autonomous Vehicle Simulation and Benchmarking`, NeurIPS 2024 / arXiv:2406.15349。成熟度：已落地仿真基准。
- Caesar et al., `nuPlan: A closed-loop ML-based planning benchmark for autonomous vehicles`, arXiv:2106.11810。成熟度：已落地闭环规划基准。
- NVIDIA, `Cosmos World Foundation Model Platform for Physical AI`, arXiv:2501.03575。成熟度：2025 产业平台。
