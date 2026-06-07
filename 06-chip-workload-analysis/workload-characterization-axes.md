# Workload Characterization Axes

上级：[Chip Workload Analysis](README.md)
相关：[Workload Analysis Methodology](workload-analysis-methodology.md), [Workload Lens](../00-overview/workload-lens.md)

## 这页在回答什么问题

这页定义整份 wiki 统一使用的 workload 刻画维度。后续每篇算法文档的 `Workload Characterization` 小节，以及 06 章节每篇 workload 深度分析，都必须沿用这些维度，避免每篇文章用不同口径描述“算力大、访存重、实时性强”。

## 统一模板

每个 workload 至少填写下面这张表。01-05 的算法文档可以用较短表述，06 的 workload 文档需要按 stage 拆开填写。

| 维度 | 必填问题 | 输出形式 |
| --- | --- | --- |
| 计算密度 | 每搬运 1 Byte 数据能产生多少有效计算 | FLOPs/Byte 量级；compute-bound / memory-bound / latency-bound / capacity-bound，并给原因 |
| 访存模式 | 数据访问是连续、strided、gather/scatter、稀疏还是随机 | 对 SRAM、DRAM row locality、DMA、cache、NoC 的影响 |
| 并行性 | 可以在哪些维度并行，哪些维度会受依赖限制 | data / channel / token / camera / candidate / horizon / pipeline parallel |
| 数据复用 | weight、activation、state、cache 能否留在片上复用 | on-chip buffer 需求和复用失败原因 |
| 量化敏感度 | 哪些算子/张量适合低比特，哪些需要高精度 | INT8 / FP8 / INT4 / FP16 / BF16 的可行性判断 |
| 瓶颈类型 | 真实部署下最先爆炸的是哪类资源 | compute、bandwidth、capacity、latency、control overhead、synchronization |
| 硬件核心需求 | 芯片必须提供什么能力才能高效执行 | MAC array、SRAM、DMA、NoC、sparse、scatter/gather、stateful cache、quantized datapath |
| archax 建模 | 06 章节额外填写，说明如何落到 Resource/Topology/Interaction/Capability | 可建模参数、资源约束、交互路径、能力开关 |

01-05 的算法文档使用前七个基础维度即可，最多在“对硬件的核心需求”里提示对应的 06 workload 文档。archax 的 Resource/Topology/Interaction/Capability 显式建模只放在 06 章节，避免算法章节变成架构建模文档。

## 计算密度

计算密度，Arithmetic Intensity，描述的是单位数据搬运量能产生多少计算。它不是简单的 FLOPs 总量。一个 workload 可以 FLOPs 很高但计算密度也高，因此适合 NPU；另一个 workload 可以 FLOPs 不高但每次计算都要重新读数据，因此受 memory bandwidth 限制。

判断计算密度时要按 stage 分析。能估算时给出 FLOPs/Byte 或高/中/低的量级锚点；如果缺少模型尺寸、输入尺寸或部署 batch，必须说明“不确定的原因”，不能只用“计算量大”替代判断。

| Stage | 常见判断 | 原因 |
| --- | --- | --- |
| CNN standard conv | 通常偏 compute-bound | weight 和 activation 都有空间复用，tile 后可在 SRAM 中复用 |
| depthwise conv | 通常偏 memory-bound | 每个 channel 独立，MAC 数下降但 activation 读写没有等比例下降 |
| Transformer prefill | 通常偏 compute-bound | QKV、FFN 是大矩阵乘，batch/token 并行度高 |
| Transformer decode | 通常偏 memory-bandwidth-bound | 每生成一个 token 要读权重和 KV cache，计算量相对小 |
| BEV view transform | 通常偏 memory/latency-bound | gather/scatter 和 sampling 破坏连续访问 |
| Occupancy dense decode | 通常偏 capacity-bound | 3D/4D tensor 随空间分辨率和 horizon 放大 |
| Diffusion / rollout | 通常偏 latency-bound 或 compute-bound | 多步循环把单步成本乘以 step count |
| SSM / selective scan（Mamba、hybrid） | 通常偏 memory / scan-kernel-bound | 状态按序列递推，associative scan 的访存像树形归约而非规则矩阵分块；状态大小恒定、不随序列长度增长，是相对 KV cache 的根本省法 |

06 文档需要避免只写“计算密集”。更好的写法是：这个 stage 的矩阵规模是否足够大，是否能形成高 MAC utilization，activation 和 weight 是否能在 SRAM 里复用，是否存在小矩阵、短序列或动态 shape 导致利用率下降。

## 访存模式

访存模式决定 RAM、DMA、cache 和 NoC 的压力。对 deterministic NPU 来说，访存模式比 FLOPs 更容易决定 p99 latency。

| 访存模式 | 典型 workload | 架构含义 |
| --- | --- | --- |
| 连续访问 | GEMM、1x1 conv、FFN | 适合 burst DMA、row locality 好、tiling 清晰 |
| 规则 strided | standard conv、FPN resize | 可预取，但需要处理 halo、stride 和边界 |
| gather/scatter | BEV lift-splat、deformable attention、point sampling | DMA 描述复杂，cache miss 和 row miss 风险高 |
| 稀疏访问 | LiDAR sparse voxel、sparse occupancy、token pruning | 需要元数据、索引、压缩格式和负载均衡 |
| 状态访问 | KV cache、temporal BEV cache、latent state | 容量和带宽随时间/长度增长，需要 cache residency 策略 |
| 小算子穿插 | norm、activation、reshape、transpose | FLOPs 小但可能造成额外搬运和同步点 |

判断访存时要问三个问题：数据是否连续，是否可预测，是否可复用。连续且可复用的数据适合 NPU tile；不连续但可预测的数据适合 DMA scatter-gather 或预计算索引；不连续且动态的数据会拉低硬件利用率，并增加 runtime 控制复杂度。

与硬件 wiki 的连接应具体到：

1. RAM：row locality、bank parallelism、SRAM/DRAM 层次是否适合该访问模式。
2. DMA：是否需要 2D/3D stride、scatter-gather、double buffering、descriptor chaining。
3. NOC：tile 之间是否有 multicast、broadcast、reduction、cross-tile state sharing。
4. PCIE/host boundary：是否存在 CPU/NPU 往返、host-side postprocess、仿真数据回传。
5. CIM：该 workload 是否有规则、大块、可复用的矩阵/卷积计算可以映射到 compute-in-memory；如果主瓶颈是 gather/scatter、状态缓存或动态控制流，要说明为什么 CIM 不是主要收益点。

## 并行性

并行性不是“能不能并行”这么简单，而是哪些维度并行后不会破坏数据依赖，哪些维度并行会增加通信。

| 并行维度 | 适用 workload | 风险 |
| --- | --- | --- |
| batch parallel | 云端推理、训练、离线仿真 | 端侧小 batch 不成立 |
| channel parallel | CNN、ConvNeXt、部分 BEV decoder | 需要足够 channel 数和 SRAM 容量 |
| token parallel | Transformer prefill、ViT、VLM encoder | decode 阶段受自回归依赖限制 |
| head parallel | multi-head attention | head 间并行容易，后续 concat/projection 仍需同步 |
| camera parallel | multi-camera AD perception | 融合 stage 需要跨 camera 对齐和通信 |
| candidate parallel | planning、World Model rollout | candidate 多时吞吐好，但 state/cache 容量上升 |
| horizon parallel | 部分 trajectory / rollout evaluation | 自回归 rollout 可能不能完全并行 |
| pipeline parallel | sensor encoder 到 planner 多 stage | 同步点和 buffer 管理复杂 |

端侧 workload 经常不是缺少理论并行性，而是缺少能被硬件稳定吃满的并行性。robot action policy、Transformer decode、short horizon planning 都可能因为 batch 小、step 依赖强、控制频率高而变成 latency-bound。

## 数据复用

数据复用决定 on-chip buffer 是否有效。复用可以来自 weight、activation、state、temporal history 或多任务共享 encoder。

| 复用类型 | 典型场景 | 对硬件的要求 |
| --- | --- | --- |
| weight reuse | CNN conv、GEMM、FFN | weight tiling、weight stationary / output stationary 策略 |
| activation reuse | conv sliding window、attention Q/K/V block | SRAM tile、halo buffer、online softmax |
| multi-task reuse | AD shared encoder、BEV shared feature | producer-consumer buffer、避免重复编码 |
| temporal reuse | video history、BEV temporal cache、track memory | state residency、cache update 和失效策略 |
| KV reuse | Transformer decode | KV cache layout、bandwidth、capacity |
| rollout state reuse | World Model candidate rollout | candidate state layout、early exit、state checkpoint |

常见误解：模型参数越大，复用越好。实际上，参数大只说明 weight 读写压力大；复用是否好取决于计算是否能在同一 tile 上反复使用这些 weight，以及 activation 是否能留在片上。decode 阶段即使参数固定，也可能因为 batch 小和 KV 读写导致 memory-bound。

## 量化敏感度

量化敏感度要按张量和 stage 写。不能把一个模型简单标成“支持 INT8”。

| 部分 | 常见低比特可行性 | 风险 |
| --- | --- | --- |
| CNN conv / pointwise conv | INT8 通常成熟 | depthwise、small channel、后处理可能限制收益 |
| Transformer GEMM / FFN | INT8/FP8 可行性较高 | activation outlier、LayerNorm/RMSNorm、softmax 需要处理 |
| Attention score / softmax | 通常需要更谨慎 | 数值范围和归一化影响稳定性 |
| KV cache | INT8/FP8/低比特压缩有价值 | 长上下文误差累积和精度退化 |
| BEV sampling / geometry transform | 不宜只看低比特 MAC | 坐标、插值、index 精度影响空间对齐 |
| Occupancy logits / sparse metadata | logits 可量化，metadata 不等价于 tensor | 稀疏索引和压缩格式不是普通低比特算子 |
| Diffusion denoising | 低比特收益受 step 和误差累积影响 | 多步采样可能放大量化误差 |
| Action head / control output | 通常需要谨慎 | 动作小误差会进入闭环并累积 |

06 文档应说明量化改变的是哪个瓶颈。如果 workload 原本 compute-bound，低比特可能直接提升吞吐和能效；如果 workload 原本 memory-bound，低比特可能通过减小数据搬运有效；如果瓶颈是 irregular access 或 CPU/NPU sync，低比特可能只能改善局部算子，不能解决系统瓶颈。

## 瓶颈类型

瓶颈类型必须明确到部署条件和 stage。建议使用下面的标签：

| 标签 | 含义 | 典型场景 |
| --- | --- | --- |
| compute-bound | MAC array 吞吐是第一瓶颈 | large conv、large GEMM、Transformer prefill |
| memory-bandwidth-bound | 数据读写速度是第一瓶颈 | KV cache decode、depthwise conv、large feature fusion |
| memory-capacity-bound | 状态或中间张量放不下 | long context、3D occupancy、multi-frame BEV |
| latency-bound | 单次调用实时预算最紧 | robot policy、AD planning、small batch decode |
| irregular-access-bound | 索引访问导致 row/cache/DMA 效率低 | BEV sampling、LiDAR sparse voxel |
| synchronization-bound | stage 间等待和 CPU/NPU 往返限制性能 | safety shell、postprocess、dynamic routing |
| control-overhead-bound | 动态 shape、分支、调度开销过高 | token pruning、sparse experts、candidate filtering |

一个 workload 可以有多个瓶颈，但必须按优先级写。例如：

```text
BEV view transform:
第一瓶颈是 irregular-access-bound，因为 camera feature 到 BEV grid 的采样/散射破坏连续访问；
第二瓶颈是 memory-bandwidth-bound，因为多相机、多尺度 feature 需要跨 stage 搬运；
compute 不是第一瓶颈，因为该 stage 的 MAC 量不一定最高。
```

## 硬件核心需求

硬件需求应从 workload 推导出来，不应直接写成“需要高 TOPS”。建议使用下面的需求语言：

| 需求类型 | 具体描述 |
| --- | --- |
| Compute | 需要高利用率 Conv/GEMM array，支持 INT8/FP8/混合精度 |
| On-chip memory | 需要足够 SRAM 保存 tile、activation、KV/BEV/latent state |
| External memory | 需要高带宽 DRAM/HBM，或通过压缩/低比特降低 bandwidth |
| DMA | 需要 2D/3D stride、scatter-gather、double buffering、descriptor chaining |
| NOC | 需要 tile 间 multicast、reduction、shared state access、QoS |
| Sparse support | 需要 sparse metadata decode、indexed load/store、负载均衡 |
| Stateful execution | 需要 cache residency、state update、rollout loop、decode loop、SSM/selective-scan 的状态递推 |
| Quantized datapath | 需要 INT8/FP8/INT4 compute 和高精度 accumulation |
| CIM suitability | 判断规则 GEMM/Conv 是否适合 CIM，以及访存/稀疏/动态 stage 为什么不适合 |
| Runtime determinism | 需要 bounded latency、固定调度、减少 CPU/NPU sync |

## archax 建模口径

每篇 06 workload 文档都应把上面的维度映射到 archax：

| archax 轴 | 应描述的内容 | 示例 |
| --- | --- | --- |
| Resource | 资源容量与吞吐 | MAC TOPS、SRAM MB、DRAM GB/s、DMA channel、NoC bandwidth |
| Topology | 资源连接方式 | NPU tile mesh、shared SRAM、memory hierarchy、host-device boundary |
| Interaction | 数据搬运和执行关系 | encoder -> BEV -> planner、KV cache update、candidate rollout loop |
| Capability | 可支持的算子和执行模式 | Conv/GEMM/Attention、scatter-gather、sparse、quantization、stateful decode、SSM/selective-scan state recurrence、hybrid attention-SSM |

archax 建模时应把 workload 参数显式化。例如 Transformer 不是一个固定 workload，而是由 `sequence length`、`batch`、`hidden size`、`layer count`、`KV cache layout`、`prefill/decode ratio` 定义的一族 workload。BEV 不是一个固定 workload，而是由 `camera count`、`image resolution`、`feature scale`、`BEV grid`、`history window`、`sampling pattern` 定义的一族 workload。

## 算法文档的简版填写格式

01-05 算法文档的 `Workload Characterization` 小节建议使用下面格式：

```text
## Workload Characterization

- 计算密度：说明主要 stage 是 compute-bound 还是 memory-bound，并给原因。
- 访存模式：说明访问是否连续，是否存在 gather/scatter、稀疏或状态缓存。
- 并行性：说明可并行维度和被依赖限制的维度。
- 数据复用：说明 weight / activation / state / temporal cache 的复用机会。
- 量化敏感度：说明哪些部分适合低比特，哪些需要高精度。
- 瓶颈类型：给出第一瓶颈和第二瓶颈。
- 对硬件的核心需求：用一两句话说明最需要算力、带宽、容量、低延迟还是特定算子支持。
```

06 workload 文档不应只用这份简版，而应按 stage 展开，并增加典型形状、端云差异、硬件连接和 archax 建模。

## 一句话理解

Workload characterization 的价值在于把“模型结构”变成“芯片约束”：每个算法最终都要落到计算密度、访存模式、并行性、复用、量化、瓶颈和硬件能力这几个可比较维度。

## Workload Characterization

这页定义的是元维度，不是单一 workload。后续具体文档必须使用这里的模板；如果某个 workload 需要新增维度，应先说明为什么现有维度无法表达，再更新本页。

## 参考来源

- Williams et al., `Roofline: An Insightful Visual Performance Model for Multicore Architectures`, CACM 2009。成熟度：已落地，计算密度/瓶颈维度的方法论来源。
- 本 wiki [Workload Analysis Methodology](workload-analysis-methodology.md)，五步分析流程。
