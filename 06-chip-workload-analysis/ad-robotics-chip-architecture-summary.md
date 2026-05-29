# AD and Robotics Chip Architecture Summary

上级：[Chip Workload Analysis](README.md)
相关：[Edge Inference Chip Requirements](edge-inference-chip-requirements.md), [Cloud Inference and Simulation Chip](cloud-inference-and-simulation-chip.md)

## 这页在回答什么问题

这页收束自动驾驶与机器人 workload 对芯片架构的共同要求。CNN、Transformer、BEV、Occupancy、E2E、VLA、World Model 不是彼此独立的模型集合，而是一组会在端侧和云端组合出现的系统 workload。

## 总体结论

自动驾驶和机器人芯片不能只按“CNN TOPS”或“Transformer TOPS”设计。真实系统同时需要：

- 规则 dense compute：CNN、GEMM、FFN、ViT/VLM encoder。
- 不规则数据搬运：BEV projection、sparse voxel、point sampling、trajectory query。
- 状态缓存：KV cache、BEV temporal cache、occupancy future、latent rollout、robot action history。
- 闭环低延迟：planning、action decode、safety shell、controller。
- 云端扩展：world model training、simulation、scenario mining、data closed loop。

## Workload 到架构的映射

| Workload | 第一瓶颈 | 芯片设计重点 |
| --- | --- | --- |
| CNN | compute / activation bandwidth | Conv/GEMM array、SRAM tiling、INT8 |
| Transformer prefill | compute | GEMM、attention tiling、FP8/INT8 |
| Transformer decode | KV bandwidth / latency | cache layout、low batch decode |
| BEV | irregular access | scatter-gather DMA、BEV cache、NoC QoS |
| Occupancy | capacity / sparse access | 3D tiling、sparse metadata、voxel cache |
| E2E AD | p99 latency / sync | deterministic pipeline、safety near compute |
| VLA | memory capacity / action latency | VLM low batch、KV cache、action decoder |
| World Model | rollout loop / generation cost | candidate parallel、state reuse、edge-cloud split |

## 对 RAM / DMA / NOC / CIM / PCIE 的要求

RAM 必须同时服务规则 tensor 和长期 state。SRAM 不只是 Conv tile buffer，也要容纳 KV、BEV、latent、occupancy 等状态；DRAM/HBM 带宽必须按持续流式和 cache replay 建模。

DMA 是数据搬运优先原则的核心。规则 Conv/GEMM 需要 burst 和 double buffer；BEV/occupancy 需要 scatter-gather、3D stride、descriptor chaining；端侧 pipeline 需要 deadline-aware 搬运。

NOC 不能只按平均带宽设计。多相机融合、multi-head 复用、候选 rollout 和 shared state 会造成阶段性拥塞，因此需要 QoS、multicast/reduction、tile 间状态访问路径。

CIM 适合规则、大块、可复用的矩阵/卷积计算，例如 Conv、1x1、FFN、QKV projection。CIM 不能替代 DMA/NOC/RAM 对 irregular、stateful、control-heavy stage 的支持。

PCIE/host boundary 在端侧应尽量退出实时路径，在云端则是 storage、NIC、多设备调度和仿真数据回传的关键边界。

## archax 总结

- Resource：把 compute、SRAM、DRAM/HBM、DMA、NoC、CIM、host interface、CPU/DSP safety resource 都建模为资源。
- Topology：描述 NPU tile、memory hierarchy、sensor ingress、controller interface、cloud fabric 和 host boundary。
- Interaction：重点建模 tensor transfer、state update、decode loop、rollout loop、producer-consumer buffer、CPU/NPU sync。
- Capability：描述 Conv/GEMM/Attention、scatter-gather、sparse metadata、3D tile、quantization、stateful cache、deterministic scheduling。

archax 的探索空间不应只扫 TOPS。更有价值的扫描变量包括 SRAM 容量、DMA capability、NoC QoS、KV/BEV cache residency、scatter-gather 支持、低比特模式、candidate rollout 并发度和 CPU/NPU 同步路径。

## Workload Characterization

- 计算密度：系统中 dense stage 高，irregular/stateful stage 低；整体瓶颈取决于 stage composition。
- 访存模式：连续、strided、sparse、gather/scatter、state cache 同时出现，必须按 stage 建模。
- 并行性：端侧 batch 弱但 camera/head/tile/candidate 可并行；云端 batch/scenario/sample 并行强。
- 数据复用：共享 encoder、BEV/KV/latent state、condition cache 是架构收益关键。
- 量化敏感度：低比特必须分 tensor/stage 决策；安全、几何、动作和边界输出需要保守。
- 瓶颈类型：端侧以 latency/bandwidth/capacity/sync 为主；云端以 compute/capacity/IO/communication 为主。
- 硬件核心需求：数据搬运优先、stateful execution、deterministic runtime、dense compute 与 irregular access 并重。

## 参考来源

- `/mnt/e/workload-wiki/task.md`，archax 与硬件 wiki 连接要求。
- `/mnt/e/workload-wiki-old/06_芯片架构Workload分析/自动驾驶与机器人芯片架构总结.md`。
