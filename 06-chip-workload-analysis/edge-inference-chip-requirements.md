# Edge Inference Chip Requirements

上级：[Chip Workload Analysis](README.md)
相关：[E2E Workload](e2e-workload.md), [VLA Workload](vla-workload.md), [AD and Robotics Chip Architecture Summary](ad-robotics-chip-architecture-summary.md)

## 这页在回答什么问题

这页从 workload 反推端侧推理芯片需求。端侧包括车端自动驾驶 SoC、机器人本体计算和边缘 NPU，它们共同的约束是：batch 小（常为 1）、功耗固定、强实时、输入持续流式到达。这套约束意味着不能用云端"大 batch 换吞吐"的思路设计，端侧芯片的成败标准是 bounded latency 而不是峰值 TOPS。

## 为什么端侧不能只看 TOPS

高 TOPS 只说明峰值矩阵算力。但端侧 workload 的实际瓶颈往往落在别处：多路相机 ingress 的持续带宽、BEV view transform 的 gather/scatter、KV cache 的带宽、3D occupancy 的容量、CPU/NPU 同步、以及 action decode 的 p99 延迟。这些都不是"算力"能解决的。

举一个能戳破 TOPS 迷思的例子：一颗标称 254 TOPS 的车端芯片（Orin 量级）跑 BEV 感知，如果 scatter-gather DMA 能力不足，view transform 阶段就会因为 row miss 和 DMA 排队让整条流水停顿，实测利用率可能只有峰值的零头——算力再高也喂不进去。同理，端侧跑一个量化后的 VLA，瓶颈是几 B 权重能否放进内存、视觉 KV cache 的带宽够不够，而不是 MAC 阵列峰值。所以一份只给 TOPS、不给 SRAM 容量、DRAM 带宽、DMA 能力、NOC QoS 和 runtime determinism 的芯片规格，几乎无法判断它能否部署目标 workload。

## 端侧需求的六个层次

把端侧需求按"会先撑爆哪类资源"分层，比按算子分类更有用。compute 层问的是能否高利用率地跑 Conv/GEMM/Attention（对应 CNN、Transformer prefill、VLM encoder）。memory 层问的是 state/cache 放不放得下、带宽够不够（对应 KV cache、BEV cache、occupancy、VLA 权重）。data movement 层问的是 sensor 和中间张量能否稳定搬运（对应 BEV、多相机、FPN、多 head）。real-time 层问的是 p99 latency 是否有界（对应 E2E、robot action）。heterogeneity 层问的是 CPU/DSP/NPU/safety island 能否可靠同步（对应 planner、controller、postprocess）。determinism 层问的是 dynamic shape 和稀疏是否可控（对应 sparse occupancy、token decode、candidate selection）。一颗端侧芯片要同时在这六层都不出现短板，因为木桶效应下任何一层的瓶颈都会拖垮 worst-case 延迟。

## 流式与确定性：端侧最难的两件事

端侧 sensor 是持续流式到达的（相机 30 Hz、LiDAR 10 Hz），不是离线数据集。这要求 DMA 能稳定承接多路 ingress 并与计算 double buffer 重叠，任何一帧的搬运抖动都可能击穿控制周期。而 deterministic NPU 的核心诉求是：worst-case latency 可预测。dynamic shape（token pruning、稀疏 voxel、动态候选数）会让调度和访存不可预测，这对追求 bounded latency 的安全系统是直接风险——这也是为什么端侧架构经常宁可用 dense、规则但"浪费一点算力"的方案，换取确定性。

## 硬件连接

RAM：需要足够 SRAM 保存 tile 和关键 state（BEV/KV/latent），DRAM 要支持持续流式访问和 cache replay 带宽（见 RAM wiki）。DMA：必须支持多路 sensor、2D/3D stride、scatter-gather、double buffering、descriptor chaining，这是端侧"数据搬运优先"的落点（见 DMA wiki）。NOC：需要 tile 间 feature 共享、multi-head producer-consumer、以及 QoS——必须保证低优先级搬运不会阻塞控制路径（见 NOC wiki）。CIM：适合规则 Conv/GEMM/FFN，但端侧仍需传统 SRAM/NOC/DMA 处理不规则和 stateful stage，CIM 不能独立成系统。PCIE/host：端侧安全路径应尽量本地化，host 往返只用于非实时管理或低频任务。

## archax 建模

Resource：TOPS、SRAM MB、DRAM GB/s、DMA engine 数、NoC GB/s、CPU/DSP、host interface——其中端侧最该精细建模的是 SRAM 容量和 DMA 能力，而非 TOPS。Topology：sensor ingress、NPU tile mesh、shared memory、safety island、controller interface。Interaction：streaming sensor pipeline、state cache 更新、deadline-aware task graph、CPU/NPU sync。Capability：INT8/FP8/INT4、Conv/GEMM/Attention、scatter-gather、sparse metadata、stateful decode、deterministic scheduling。archax 探索端侧时，目标函数应是 worst-case latency 与功耗，而不是平均吞吐。

## 一句话理解

端侧芯片在 batch=1、固定功耗、强实时下，瓶颈几乎总是带宽、容量、不规则访问和同步，而非峰值算力；好的端侧架构是把 worst-case 延迟做有界，而不是把 TOPS 做大。

## Workload Characterization

- 计算密度：端侧混合 compute-bound 与 memory-bound，峰值 TOPS 只覆盖规则 dense stage。
- 访存模式：流式 sensor、state cache、irregular BEV、KV cache、occupancy grid 同时存在。
- 并行性：camera/head/tile/candidate 可并行；batch 并行弱，控制 loop 串行强。
- 数据复用：SRAM 中保留 BEV/KV/latent state 是关键；多 head 共享 encoder 避免重复计算。
- 量化敏感度：低比特是端侧必要条件，但 safety margin、几何、动作输出需保守。
- 瓶颈类型：p99 latency、bandwidth、capacity、synchronization 往往比平均 TOPS 更关键。
- 对硬件的核心需求：可预测低延迟、多级缓存、强 DMA、QoS NOC、低比特 dense compute、近端 safety/control。

## 参考来源

- Lin et al., `AWQ: Activation-aware Weight Quantization for LLM Compression and Acceleration`, MLSys 2024 / arXiv:2306.00978。成熟度：已落地，端侧低比特部署关键技术。
- Frantar et al., `GPTQ: Accurate Post-Training Quantization for Generative Pre-trained Transformers`, ICLR 2023 / arXiv:2210.17323。成熟度：已落地权重量化方法。
- Reuther et al., `AI Accelerator Survey and Trends`, IEEE HPEC（系列综述）。成熟度：已落地，端侧/边缘加速器 TOPS-功耗权衡参考。
