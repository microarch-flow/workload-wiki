# AD and Robotics Chip Architecture Summary

上级：[Chip Workload Analysis](README.md)
相关：[Edge Inference Chip Requirements](edge-inference-chip-requirements.md), [Cloud Inference and Simulation Chip](cloud-inference-and-simulation-chip.md), [Workload Analysis Methodology](workload-analysis-methodology.md)

## 这页在回答什么问题

这页是整份 wiki 的收尾。它把 CNN、Transformer、BEV、Occupancy、E2E、VLA、World Model 这些 workload 收束成一句话能用的架构结论，并显式连接 BUS/RAM/NOC/DMA/FAB/CIM/PCIE 七份硬件 wiki 和 archax 方法论。前面每一篇讲的是单个 workload，这一篇讲它们**组合在一颗真实芯片上**时的共同要求。

## 核心结论：真实系统是 workload 的组合，不是单一 workload

自动驾驶和机器人芯片不能按"CNN TOPS"或"Transformer TOPS"设计，因为真实系统在同一时刻、同一颗芯片上同时跑五类性质不同的 workload：规则 dense compute（CNN、GEMM、FFN、ViT/VLM encoder），不规则数据搬运（BEV projection、sparse voxel、point sampling、trajectory query），状态缓存（KV cache、BEV temporal cache、occupancy future、latent rollout、action history），闭环低延迟（planning、action decode、safety shell、controller），以及云端扩展（World Model 训练、仿真、场景挖掘、数据闭环）。

一颗只擅长其中一类的芯片会在系统集成时暴露短板。只堆 dense 算力的芯片会被 BEV 的不规则访问和 CPU/NPU 同步拖垮；只优化大模型推理的芯片会缺少 scatter-gather 和 deterministic 调度。所以架构探索的第一原则是：按 stage 组合建模，而不是按单一峰值指标选型。

## Workload 到架构的映射表

下面这张表是整份 wiki 的浓缩——每个 workload 的第一瓶颈不同，因此芯片设计的着力点也不同。这张表也是 archax 设定探索目标时的入口。

| Workload | 第一瓶颈 | 芯片设计重点 |
| --- | --- | --- |
| CNN（standard/1×1） | compute / 早层 activation 带宽 | Conv/GEMM array、SRAM tiling、INT8 |
| CNN（depthwise） | memory bandwidth | DRAM 带宽、activation 调度 |
| Transformer prefill | compute | 大 GEMM、attention tiling、FP8/INT8 |
| Transformer decode | KV bandwidth / latency | KV cache 布局与量化、低 batch decode |
| BEV | irregular access | scatter-gather DMA、BEV cache 常驻、NOC QoS |
| Occupancy | capacity / sparse access | 3D tiling、sparse metadata、voxel cache |
| E2E AD | p99 latency / 同步 | deterministic pipeline、safety 近端 |
| VLA | memory capacity / action latency | 低 batch VLM、视觉 KV cache、action decoder |
| World Model | rollout 循环 / 生成成本 | candidate 并发、condition 复用、端云分工 |

## 对七份硬件 wiki 的要求

RAM 必须同时服务规则张量和长期状态。SRAM 不只是 conv tile buffer，还要容纳 KV、BEV、latent、occupancy 这些跨时间存活的状态；DRAM/HBM 带宽要按持续流式访问和 cache replay 建模，而不是按一次性读取。BEV view transform 的 row miss 和 occupancy 的容量爆炸，都是 RAM wiki 里 row locality 与 bank parallelism 分析的直接落点。

DMA 是"数据搬运优先"原则的核心执行者。规则 Conv/GEMM 要 burst + double buffer；BEV/occupancy 要 scatter-gather、3D stride、descriptor chaining；端侧流式 pipeline 要 deadline-aware 搬运。一颗芯片的 DMA 能力短板，往往比算力短板更早决定它能不能部署 BEV/occupancy。

NOC 不能只按平均带宽设计。多相机融合、多 head 复用、候选 rollout、shared state 会造成阶段性拥塞尖峰，因此需要 QoS、multicast/reduction、tile 间状态访问路径，确保控制路径不被低优先级流量阻塞。

CIM 适合规则、大块、可复用的矩阵/卷积（Conv、1×1、FFN、QKV projection），但它不能替代 RAM/DMA/NOC 对 irregular、stateful、control-heavy stage 的支持——CIM 是 dense compute 的加速器，不是系统。

BUS 与 FAB 在系统层面决定多 NPU tile、CPU、安全岛、传感器接口之间的连接带宽与一致性；PCIE/host boundary 在端侧应退出实时路径，在云端则是 storage、NIC、多设备调度和仿真数据回传的关键边界。

## archax 总结：探索空间不该只扫 TOPS

把 compute array、SRAM、DRAM/HBM、DMA engine、NoC link、host interface、CIM array、CPU/DSP safety resource 全部建模为 Resource；用 Topology 描述 NPU tile mesh、memory hierarchy、sensor ingress、controller interface、cloud fabric 和 host boundary 怎么连；用 Interaction 重点建模 tensor transfer、state update、decode loop、rollout loop、producer-consumer buffer 和 CPU/NPU sync——其中同步点和循环要建成有成本的交互；用 Capability 描述 Conv/GEMM/Attention、scatter-gather、sparse metadata、3D tile、quantization、stateful cache 和 deterministic scheduling。

archax 探索这些 workload 时，比 TOPS 更有价值的扫描变量是：SRAM 容量、DMA capability、NOC QoS、KV/BEV cache residency、scatter-gather 支持、低比特模式、candidate rollout 并发度，以及 CPU/NPU 同步路径数。这正是这份 workload wiki 与七份硬件 wiki 的接缝——前者告诉你 workload 会先撑爆哪类资源，后者告诉你那类资源该怎么设计。

## 一句话理解

自动驾驶与机器人芯片的本质是承载一组性质相反的 workload 组合，架构成败取决于 dense compute、irregular access、stateful cache、bounded latency 四者的平衡，而 archax 的价值是把这个平衡变成可扫描、可比较的探索空间——而不是一个 TOPS 数字。

## Workload Characterization

- 计算密度：系统中 dense stage 高、irregular/stateful stage 低；整体瓶颈取决于 stage composition 而非单一指标。
- 访存模式：连续、strided、sparse、gather/scatter、state cache 同时出现，必须按 stage 建模。
- 并行性：端侧 batch 弱但 camera/head/tile/candidate 可并行；云端 batch/scenario/sample 并行强。
- 数据复用：共享 encoder、BEV/KV/latent state、condition cache 是架构收益的关键来源。
- 量化敏感度：低比特必须分 tensor/stage 决策；安全、几何、动作和边界输出需保守。
- 瓶颈类型：端侧以 latency/bandwidth/capacity/sync 为主；云端以 compute/capacity/IO/communication 为主。
- 对硬件的核心需求：数据搬运优先、stateful execution、deterministic runtime，dense compute 与 irregular access 并重。

## 参考来源

- 本 wiki 06 章各 workload 篇，以及 [Workload Characterization Axes](workload-characterization-axes.md) 定义的统一维度。
- 配套硬件 wiki：BUS、RAM、NOC、DMA、FAB、CIM、PCIE（archax 探索的硬件侧依据）。
- Sze et al., `Efficient Processing of Deep Neural Networks: A Tutorial and Survey`, Proceedings of the IEEE 2017 / arXiv:1703.09039。成熟度：已落地，workload 到硬件需求映射的经典综述。
