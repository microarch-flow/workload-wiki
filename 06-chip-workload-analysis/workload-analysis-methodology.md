# Workload Analysis Methodology

上级：[Chip Workload Analysis](README.md)
相关：[Workload Characterization Axes](workload-characterization-axes.md), [Workload Lens](../00-overview/workload-lens.md)

## 这页在回答什么问题

这页回答一个问题：如何把一个算法或模型，从论文里的 network architecture 翻译成芯片架构探索里可建模、可比较、可落地的 workload 描述。这里不追求复现算法训练细节，而是建立一套稳定流程，用来判断计算结构、访存行为、并行性、数据复用、量化敏感度和硬件需求。

## 方法论的边界

这份 wiki 的 workload 分析面向 deterministic NPU 和端侧/车端/机器人推理，不是面向通用 GPU 训练优化。训练 workload 会在必要时出现，例如 World Model 的云端仿真训练、VLA 的大规模 imitation learning、Diffusion 的 denoising training；但 06 的主线仍然是把在线推理和仿真推理映射到芯片架构需求。

算法文档回答“模型为什么这样设计”，workload 文档回答“这个设计给芯片带来了什么压力”。同一个模型在算法视角下可能是一个 unified architecture，在 workload 视角下必须拆成多个 stage，因为不同 stage 的瓶颈可能完全不同。Transformer 就是典型例子：prefill 是大矩阵乘主导，decode 是 KV cache 读写主导；如果把它们写成一个 workload，就会把算力需求和带宽需求混在一起，得到错误的芯片结论。

因此，本章采用的基本原则是：

1. 先拆数据流，再谈算子。
2. 先判断瓶颈，再谈优化。
3. 先区分端侧、云端、仿真，再比较模型。
4. 先给可建模维度，再给架构建议。

## 从算法到 workload 的五步流程

### 1. 抽象输入、输出和状态

第一步不是列论文模块，而是写清楚系统边界：

| 问题 | 需要记录的内容 |
| --- | --- |
| 输入是什么 | camera、LiDAR、history frames、language prompt、robot proprioception、map prior |
| 输出是什么 | class / box / mask、BEV feature、occupancy、trajectory、action chunk、future latent、risk score |
| 状态是什么 | KV cache、temporal BEV cache、track memory、latent state、rollout state、robot action history |
| 调用频率是多少 | 单帧、video chunk、10 Hz policy、30 Hz perception、离线 cloud batch |
| 是否闭环 | 单步 inference、receding horizon、multi-candidate rollout、simulation loop |

这一步决定后续分析是否准确。因为 workload 的压力经常不来自单次 forward，而来自状态随时间累积。例如 KV cache 随 sequence length 增长，BEV temporal cache 随 history window 增长，World Model rollout 随 `candidate count x horizon` 增长。

常见误解：只要模型结构相同，workload 就相同。实际上，同一个 Transformer block 在 batch prefill、single-token decode、cross-modal fusion、action token decode 中的 arithmetic intensity 和访存压力都不同。

### 2. 拆成 stage，而不是只看 end-to-end graph

端侧 workload 需要按 latency-critical path 拆 stage。自动驾驶和机器人系统尤其不能只画一个大模型框，因为 sensor input、feature extraction、temporal fusion、policy decode、安全约束、controller 之间存在同步点。

建议每篇 06 文档至少拆出以下内容：

| Stage 类型 | 典型例子 | 为什么要单独拆 |
| --- | --- | --- |
| Dense compute stage | CNN backbone、ViT projection、FFN | 主要看 MAC utilization、tiling、片上复用 |
| Irregular memory stage | BEV view transform、point cloud voxelization、gather/scatter | 主要看 DMA、cache locality、NoC 分发 |
| Stateful stage | KV cache update、temporal BEV memory、track query memory | 主要看 capacity、bandwidth、状态一致性 |
| Autoregressive stage | LLM decode、action token decode | 主要看 per-token latency 和 cache bandwidth |
| Rollout stage | World Model candidate rollout、Diffusion denoising | 主要看循环次数、候选并行、早停 |
| Safety/control stage | trajectory validation、constraint checking、controller | 主要看 CPU/NPU sync 和 p99 latency |

拆 stage 的目的不是让文档变复杂，而是避免错误归因。一个端到端系统的平均 TOPS 需求可能不高，但如果其中一个 view transform 或 action decode stage 造成 p99 latency 尖峰，系统仍然不能部署。

### 3. 定义主导成本，而不是罗列所有算子

每个模型都有很多算子，但架构分析只需要抓住会改变硬件设计的部分。主导成本可以来自计算、带宽、容量、延迟、动态控制流或同步点。

| 成本来源 | 典型 workload | 判断方式 |
| --- | --- | --- |
| Compute | CNN standard conv、Transformer prefill FFN、large ViT encoder | FLOPs 高、数据复用好、矩阵规模大 |
| Memory bandwidth | Transformer decode、KV cache read、depthwise conv、large feature map fusion | FLOPs/Byte 低、反复读大状态 |
| Memory capacity | long context KV cache、3D occupancy、multi-frame BEV cache | 中间状态随 token/grid/history 线性或超线性增长 |
| Irregular access | BEV lift-splat、LiDAR sparse voxel、point sampling | gather/scatter 多，row locality 差 |
| Latency | robot action policy、AD sensor-to-control path | 小 batch、强实时、同步点多 |
| Control overhead | dynamic token pruning、sparse routing、candidate selection | shape/索引动态变化，硬件利用率不稳定 |

这一步需要输出一个明确判断：这个 workload 的第一瓶颈是什么，第二瓶颈是什么。不能只写“算力和带宽都重要”，除非进一步说明哪个 stage 受算力限制，哪个 stage 受带宽限制。

### 4. 用统一维度填写 workload characterization

所有 01-05 的算法文档和 06 的 workload 文档，都应使用 [Workload Characterization Axes](workload-characterization-axes.md) 的维度，但粒度不同：

| 文档类型 | 需要达到的粒度 |
| --- | --- |
| 01-05 算法文档 | 在文末用统一维度给出定性画像，说明该算法为什么对硬件有这种压力 |
| 06 workload 文档 | 按 stage 给出更细画像，包含典型形状、瓶颈归因、端云差异、硬件连接和 archax 建模 |
| 07 reference | 把各 workload 的维度压缩成对比表和索引 |

06 文档不能只复述算法章节。它必须回答这些问题：

1. 这个 workload 的可建模输入参数是什么，例如 token length、BEV grid、voxel resolution、candidate count、horizon、action chunk length。
2. 这些参数如何放大计算量、访存量和状态容量。
3. 哪些 stage 适合 deterministic NPU，哪些 stage 需要 CPU/DSP/专用数据搬运协同。
4. 量化、稀疏、低精度是否会改变瓶颈，还是只减少部分 compute。
5. 对 RAM、DMA、NOC、CIM、PCIE/host-device boundary 的要求分别是什么。

### 5. 映射到硬件 wiki 和 archax

06 的每篇文档最后都要落到硬件和 archax。这里的映射不是泛泛地说“需要高带宽”，而是把 workload 的行为翻译成硬件建模字段。

| Workload 观察 | 硬件 wiki 连接 | archax 抽象 |
| --- | --- | --- |
| 大块连续 activation / weight 复用 | RAM row locality、bank parallelism、SRAM tiling | Resource: SRAM/DRAM capacity and bandwidth |
| 多 stage 大量 tensor 搬运 | DMA descriptor、burst、double buffering | Interaction: producer-consumer transfer |
| 多 NPU tile 共享 feature / KV / BEV cache | NOC multicast、reduction、contention | Topology: tile graph and bandwidth edge |
| irregular gather/scatter | RAM row miss、DMA scatter-gather、cache miss | Capability: indexed load/store, sparse access |
| 低比特 GEMM/Conv | CIM / MAC array / quantized datapath | Capability: INT8/FP8/INT4 compute modes |
| stateful rollout / decode | SRAM residency、HBM bandwidth、host sync | Interaction: state update loop |
| CPU/NPU 同步点 | PCIE/host-device boundary、runtime scheduling | Topology + Interaction: heterogeneous execution path |

archax 的 Resource/Topology/Interaction/Capability 可以这样落地：

Resource 描述“有什么资源”，包括 compute array、SRAM、DRAM/HBM、DMA engine、NoC link、host interface。Topology 描述“资源怎么连”，包括 tile 间互连、memory hierarchy、端云边界。Interaction 描述“数据怎么动”，包括 tensor producer-consumer、cache update、rollout loop、CPU/NPU sync。Capability 描述“能执行什么”，包括 Conv/GEMM/Attention、scatter/gather、sparse metadata、quantization、stateful decode、online softmax。

## 典型 workload 的分析入口

| Workload | 优先拆解参数 | 第一关注点 | 06 对应文档 |
| --- | --- | --- | --- |
| CNN | feature map size、kernel、channel、stride、batch | data reuse 和 MAC utilization | [CNN Workload](cnn-workload.md) |
| Transformer | sequence length、hidden size、head count、KV cache、batch | prefill/decode 分裂 | [Transformer Workload](transformer-workload.md) |
| BEV | camera count、image resolution、BEV grid、history frames | view transform 的非规则访存 | [BEV Workload](bev-workload.md) |
| Occupancy | voxel resolution、semantic channels、temporal horizon | 3D/4D tensor capacity | [Occupancy Workload](occupancy-workload.md) |
| E2E AD | sensor rate、history window、planning horizon、safety stage | p99 latency 和多任务级联 | [E2E Workload](e2e-workload.md) |
| VLA | visual tokens、language tokens、action horizon、control frequency | action decode 和端侧实时性 | [VLA Workload](vla-workload.md) |
| World Model | candidate count、rollout horizon、latent size、denoise steps | rollout loop 和 state reuse | [World Model Workload](world-model-workload.md) |

## 端侧、云端和仿真的口径

同一个算法在不同部署位置下应写成不同 workload。

端侧/车端推理关注的是固定功耗和强实时。这里 batch 通常很小，pipeline 中存在 camera/LiDAR input、NPU inference、CPU safety shell、controller 的同步约束。端侧最怕的是看起来平均吞吐足够，但某个 stage 因为 cache miss、DMA 排队或动态 shape 造成 p99 latency 超标。

云端推理关注的是吞吐、并发和显存/内存容量。大模型 prefill、长上下文 VLM、批量仿真、离线数据挖掘都可以通过 batching 和并行调度提高利用率。云端优化不能直接迁移到端侧，因为云端可接受更高功耗、更大 batch 和更复杂 runtime。

仿真和 World Model 关注的是 rollout 成本。这里关键参数不是单帧 FLOPs，而是 `candidate count x horizon x model step cost`。如果还使用 Diffusion 或 autoregressive video model，采样步数或 token decode 会进一步放大延迟和成本。

## 常见误区

只看 FLOPs 是第一个误区。FLOPs 高的 workload 不一定难部署，CNN standard conv 和 Transformer prefill 在数据复用充分时可以很好地喂满 MAC array；FLOPs 低的 workload 也可能很难部署，例如 depthwise conv、KV cache decode、BEV gather/scatter。

把训练 workload 和推理 workload 混在一起是第二个误区。VLA 和 World Model 的训练可能是云端大 batch、大显存、大通信问题，但端侧推理可能变成 action head latency、history cache、receding horizon control 问题。

把 end-to-end 当成单一 workload 是第三个误区。E2E 自动驾驶和机器人 VLA 都是系统 workload，里面同时包含 dense vision encoder、temporal memory、token fusion、policy decode、safety/control。架构建模必须按 stage 做，不然无法解释瓶颈。

把 World Model 等同于视频生成是第四个误区。视频生成关注视觉逼真度和长序列生成，决策级 World Model 关注可控状态预测、风险评估、multi-candidate rollout 和闭环一致性。它们可能共享 Diffusion 或 Transformer 组件，但 workload 目标不同。

## 一句话理解

Workload 分析的核心不是把算子列出来，而是把算法的数据流、状态、瓶颈和部署约束翻译成 NPU 架构探索可以建模的 Resource、Topology、Interaction 和 Capability。

## Workload Characterization

这篇方法论文档本身不是某个具体 workload，但它定义所有后续文档的分析流程。具体 workload 的 characterization 必须按 [Workload Characterization Axes](workload-characterization-axes.md) 填写，并在 06 文档中进一步连接硬件 wiki 与 archax。

## 参考来源

- Williams et al., `Roofline: An Insightful Visual Performance Model for Multicore Architectures`, CACM 2009。成熟度：已落地，arithmetic intensity 与瓶颈判断的方法论基础。
- Sze et al., `Efficient Processing of Deep Neural Networks: A Tutorial and Survey`, Proceedings of the IEEE 2017 / arXiv:1703.09039。成熟度：已落地，算法到硬件需求映射的经典综述。
- 本 wiki [Workload Characterization Axes](workload-characterization-axes.md)，统一刻画维度的定义。
