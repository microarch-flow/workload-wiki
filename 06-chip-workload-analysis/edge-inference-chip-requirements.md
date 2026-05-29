# Edge Inference Chip Requirements

上级：[Chip Workload Analysis](README.md)
相关：[E2E Workload](e2e-workload.md), [VLA Workload](vla-workload.md), [AD Robotics Summary](ad-robotics-chip-architecture-summary.md)

## 这页在回答什么问题

这页从 workload 反推端侧推理芯片需求。端侧包括车端自动驾驶芯片、机器人本体计算和边缘部署 NPU。它们共同特点是 batch 小、功耗固定、实时性强、输入持续流式到达，不能用云端大 batch 吞吐思维设计。

## 需求分层

| 层次 | 核心问题 | 对应 workload |
| --- | --- | --- |
| compute | 是否能高利用率跑 Conv/GEMM/Attention | CNN、Transformer prefill、VLM encoder |
| memory | state/cache 是否放得下、带宽是否够 | KV cache、BEV cache、occupancy、VLA |
| data movement | sensor 和中间 tensor 是否稳定搬运 | BEV、多相机、FPN、多 head |
| real-time | p99 latency 是否有界 | E2E AD、robot action |
| heterogeneity | CPU/DSP/NPU/safety 是否同步可靠 | planner、controller、postprocess |
| determinism | dynamic shape 和稀疏是否可控 | sparse occupancy、token decode、candidate selection |

## 端侧不能只看 TOPS

高 TOPS 只能说明峰值矩阵计算能力。端侧 workload 的实际瓶颈经常是：多路 camera ingress、BEV gather/scatter、KV cache bandwidth、3D occupancy capacity、CPU/NPU 同步、action decode p99。芯片规格如果只给 TOPS，而没有 SRAM、DRAM bandwidth、DMA 能力、NoC QoS 和 runtime determinism，很难判断能否部署。

## 硬件连接

- RAM：需要足够 SRAM 保存 tile 和关键 state；DRAM 需要支持持续流式访问和 cache bandwidth。
- DMA：必须支持多路 sensor、2D/3D stride、scatter-gather、double buffering、descriptor chaining。
- NOC：需要 tile 间 feature 共享、multi-head producer-consumer、QoS，避免低优先级搬运阻塞控制路径。
- CIM：适合规则 Conv/GEMM/FFN；端侧仍需要传统 SRAM/NOC/DMA 处理不规则和 stateful stage。
- PCIE/host：端侧安全路径应尽量本地化；host 往返只用于非实时管理或低频任务。

## archax 建模

- Resource：TOPS、SRAM MB、DRAM GB/s、DMA engines、NoC GB/s、CPU/DSP、host interface。
- Topology：sensor ingress、NPU tile mesh、shared memory、safety island、controller interface。
- Interaction：streaming sensor pipeline、state cache update、deadline-aware task graph、CPU/NPU sync。
- Capability：INT8/FP8/INT4、Conv/GEMM/Attention、scatter-gather、sparse metadata、stateful decode、deterministic scheduling。

## Workload Characterization

- 计算密度：端侧混合 compute-bound 与 memory-bound；峰值 TOPS 只覆盖规则 dense stage。
- 访存模式：流式 sensor、state cache、irregular BEV、KV cache、occupancy grid 同时存在。
- 并行性：camera/head/tile/candidate 可并行；batch 并行弱，控制 loop 串行强。
- 数据复用：SRAM 中保留 BEV/KV/latent state 是关键；多 head 共享 encoder 避免重复计算。
- 量化敏感度：低比特是端侧必要条件，但 safety margin、geometry、action output 需要保守。
- 瓶颈类型：p99 latency、bandwidth、capacity 和 synchronization 往往比平均 TOPS 更关键。
- 硬件核心需求：可预测低延迟、多级缓存、强 DMA、QoS NoC、低比特 dense compute、近端 safety/control。

## 参考来源

- `/mnt/e/workload-wiki/task.md`，端侧 deterministic NPU 和 archax 需求。
- `/mnt/e/workload-wiki-old/06_芯片架构Workload分析/端侧推理芯片需求.md`。
