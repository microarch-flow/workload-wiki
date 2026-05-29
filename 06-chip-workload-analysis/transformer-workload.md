# Transformer Workload

上级：[Chip Workload Analysis](README.md)
相关：[Attention and Transformer](../01-foundation-model-components/attention-and-transformer.md), [Attention Variants and Efficiency](../01-foundation-model-components/attention-variants-and-efficiency.md), [VLA Workload](vla-workload.md)

## 这页在回答什么问题

这页要说清一件事：Transformer 推理不是单一 workload，而是 prefill 和 decode 两个**特征几乎相反**的阶段，理解这个分裂是理解所有 Transformer 类芯片需求的钥匙。如果把 Transformer 写成"一个计算量大、需要高算力的模型"，几乎所有后续的芯片结论都会错。

## 为什么必须先把 Transformer 拆成两个阶段

一次自回归生成由两段组成。先是 prefill：模型把整段输入 prompt 一次性读进来，所有 token 并行过一遍全部层，算出每一层的 K、V 并存入 KV cache，最后吐出第一个输出 token。然后是 decode：模型自回归地逐 token 生成，每生成一个新 token，都要把它和**之前所有 token 的 KV cache**做一遍 attention，再过一遍全部权重，得到下一个 token。

这两段跑的是同一组权重、同一个网络结构，但 arithmetic intensity 相差一两个数量级，因此对硬件提出几乎相反的要求。下面用具体数字说明。

设一个 7B 参数的 decoder-only 模型，hidden size 4096、32 层、FP16 权重，权重总量约 14 GB。

在 **prefill** 阶段，假设 prompt 长度 N=2048。QKV projection、FFN 这些大矩阵乘的左矩阵是 `[N, hidden]`、右矩阵是 `[hidden, hidden]` 量级，N 越大，同一份权重被复用的次数越多。权重只需从 DRAM 读一次（14 GB），却参与了 N 个 token 的计算，所以每读 1 字节权重能摊出 O(N) 次乘加。arithmetic intensity 高达数百 FLOP/Byte，远超现代加速器的 machine balance（典型 H100/Orin 一类器件在几十到一两百 FLOP/Byte），因此 prefill 是稳稳的 **compute-bound**：它吃的是 MAC 阵列的峰值算力，batch 和长 prompt 都让它更高效。

在 **decode** 阶段，batch=1 时每步只生成 1 个 token。要算这一个 token，仍然要把全部 14 GB 权重读一遍，外加读一遍当前长度的 KV cache，但有效计算只有"1 个 token 过一遍网络"的量。arithmetic intensity 掉到约 2 FLOP/Byte（每个权重读 2 字节、做 1 次乘加=2 FLOP），远低于 machine balance，因此 decode 是 **memory-bandwidth-bound**：瓶颈完全在把权重和 KV cache 从 DRAM 搬进来的速度。一个直接的上界估算：若 HBM 带宽 1 TB/s，读 14 GB 权重需约 14 ms，于是 batch=1 的单流解码上限约 70 token/s——不管 MAC 阵列峰值多高都救不了，因为算力根本没被喂满。

这就是 Transformer 芯片设计的核心矛盾：prefill 要算力，decode 要带宽，一颗芯片很难同时把两者做到最优。

## KV cache：同时压垮容量和带宽的状态

KV cache 是把 decode 从 compute-bound 拉成 memory-bound 的直接原因，它对硬件施加两重压力。

容量压力来自它随序列长度和 batch 线性增长。每个 token、每层都要存一份 K 和一份 V，单 token 的 KV 大小是 `2 × layers × hidden × bytes`。还是上面的 7B 模型（32 层、hidden 4096、FP16）：单 token 约 `2 × 32 × 4096 × 2 B = 512 KB`。一条 4096-token 的上下文就要 2 GB KV cache；如果要并发服务 batch=32，就是 64 GB——已经超过权重本身。长上下文（32K、128K）会把 KV cache 推成比权重更大的内存占用者，这是长上下文推理 capacity-bound 的根源。

带宽压力来自 decode 每一步都要把整个 KV cache 重新读一遍做 attention。序列越长，每个新 token 的 attention 读量越大，decode 单步延迟随上下文长度上升。所以 KV cache 既占 memory capacity，又在 decode 阶段和权重一起争 memory bandwidth——这是 LLM/VLA 推理芯片绕不开的矛盾。

常见误解：模型参数越大，越需要算力。实际上对 batch=1 decode 而言，参数越大首先意味着每步要搬越多权重字节，瓶颈是带宽不是算力；真正吃算力的是 prefill 和大 batch 服务。

## 分阶段的逐算子画像

把一个 Transformer block 再拆细，不同算子在两个阶段的角色不同：

QKV projection、output projection、FFN 的两个线性层是 block 里 FLOPs 最大的部分，都是规则大矩阵乘，prefill 时 compute-bound、适合低比特 GEMM；decode 时退化成 `[1, hidden] × [hidden, hidden]` 的瘦长矩阵乘，MAC 利用率极低，纯粹被权重带宽限制。

attention 内部的 QK^T、softmax、AV 在长序列 prefill 时是第二大成本，且 softmax 需要在线归约与数值稳定（这正是 FlashAttention 用 tiling + online softmax 解决的问题，见 [Attention Variants and Efficiency](../01-foundation-model-components/attention-variants-and-efficiency.md)）；decode 时 attention 变成新 token 对全量 KV 的访问，是 KV 带宽的主要消费者。

RMSNorm/LayerNorm、残差、reshape、transpose 这些算子 FLOPs 很小，但会在 GEMM 之间插入小算子、造成额外的 activation 搬运和同步点，在 decode 这种本就 latency 敏感的阶段，它们的固定开销占比会被放大。

## 可建模参数

archax 探索时，Transformer 不是一个固定 workload，而是由下面这组参数定义的一族 workload。把它们显式化，才能在架构空间里扫描：

`sequence length` 放大 attention 计算、KV cache 容量和 softmax IO；`hidden size` 放大 QKV/FFN 的 GEMM FLOPs 和权重体积；`layer count` 线性放大状态访问和单步延迟；`head count / head dim` 决定 attention 并行度和 KV cache 的内存布局；`batch` 决定 decode 阶段的权重复用程度（端侧 batch=1 最难，云端合批可摊薄权重读取）；`KV precision` 直接决定 cache 的带宽和容量占用；最关键的是 `prefill/decode 比例`，它决定整个负载落在 compute-bound 还是 bandwidth-bound 的哪一侧。

## 硬件连接

prefill 阶段需要大块连续的 weight/activation 搬运，对 RAM 的 row locality 和 bank parallelism 友好，DMA 适合 burst + double buffering（见 RAM、DMA wiki）。decode 阶段相反：它需要高效的 KV cache 布局和高 DRAM/HBM 带宽，DMA 退化成小块、重复、低延迟的搬运，对 row locality 的利用取决于 KV cache 是否按访问顺序连续存放——这是 RAM wiki 里 row-buffer 命中分析的直接落点，也是为什么 decode 主导的部署需要 HBM 而非仅靠大 SRAM。

多 NPU tile 协同时，attention 需要 Q/K/V 的分发、softmax 的跨 tile 归约，以及多 tile 共享同一份 KV cache 的路径，这些都落到 NOC wiki 的 multicast/reduction/QoS。FFN、QKV/output projection 是规则大 GEMM，是 CIM wiki 最适合的对象；而 softmax、KV cache 访问、动态 decode 循环不是 CIM 的主收益点，仍需传统 SRAM/DMA/NOC 支撑。云端跨设备部署时，prefill/decode 分离、tensor/pipeline 并行会让 PCIE/NVLink 边界变得关键；端侧则必须避免在 token decode 循环里出现 host 往返。

## archax 建模

Resource：GEMM TOPS、SRAM 与 KV cache 容量、DRAM/HBM 带宽、NoC reduction 带宽——其中 decode 工作点的关键资源是带宽和容量，prefill 工作点的关键资源是 TOPS。

Topology：attention head 到 tile 的映射、KV cache 所在的 memory level（SRAM 常驻还是 DRAM 换入换出）、host-device 边界。

Interaction：`prefill block`、`decode loop`、`KV cache update`、`cross-modal token fusion` 是必须显式建模的交互；其中 prefill 和 decode 在"数据搬运 vs 计算"这条轴上是两个极端点，架构探索时必须作为**两个独立工作点分别评估**，绝不能用平均值代替。

Capability：大 GEMM、batched/online-softmax attention、KV cache quantization、INT8/FP8/INT4 datapath、stateful decode。

## 一句话理解

Transformer 推理是一个会在 prefill 和 decode 之间在 compute-bound 与 memory-bandwidth-bound 之间反复横跳的 workload，KV cache 是把它拉向带宽与容量瓶颈的那只手；任何 Transformer 芯片决策都要先问"这是哪个阶段、batch 多大"。

## Workload Characterization

下面把 prefill 和 decode 两个阶段按统一维度并排刻画，差异一目了然（以 batch=1 端侧场景为基准，云端合批的变化在括号内注明）。

| 维度 | Prefill / Encoder 阶段 | Decode 阶段（batch=1） |
| --- | --- | --- |
| 计算密度 | 高，数百 FLOP/Byte，compute-bound | 极低，约 2 FLOP/Byte，memory-bandwidth-bound（合批可提升） |
| 访存模式 | 大块连续 GEMM，row locality 好 | 权重整读 + KV cache 状态访问，小块重复 |
| 并行性 | token / head / batch 三维都可并行 | 受自回归依赖限制，只能在 layer/head/GEMM 内并行 |
| 数据复用 | weight/activation 复用充分 | 权重几乎无复用（合批才有），KV cache 成为主状态 |
| 量化敏感度 | FFN/QKV 适合 INT8/FP8 | 权重低比特收益最大（直接减带宽）；KV cache 低比特有价值但需防长程误差累积 |
| 瓶颈类型 | compute-bound | memory-bandwidth-bound；长上下文叠加 capacity-bound 与 latency-bound |
| 对硬件的核心需求 | 高 TOPS、attention tiling、低比特 GEMM | 高 DRAM/HBM 带宽、KV cache 布局与压缩、低 batch 高效 decode |

两阶段需求相反，催生了 prefill/decode 分离部署、推测解码（speculative decoding）、连续批处理（continuous batching）、KV cache 量化与分页（PagedAttention）等系统级方案——它们本质上都是在调和这两个工作点对硬件的矛盾需求。

## 参考来源

- Vaswani et al., `Attention Is All You Need`, NeurIPS 2017 / arXiv:1706.03762。成熟度：已落地基础架构。
- Dao et al., `FlashAttention: Fast and Memory-Efficient Exact Attention with IO-Awareness`, NeurIPS 2022 / arXiv:2205.14135。成熟度：已落地。
- Pope et al., `Efficiently Scaling Transformer Inference`, MLSys 2023 / arXiv:2211.05102。成熟度：已落地，prefill/decode 与 KV cache 带宽分析的代表工作。
- Kwon et al., `Efficient Memory Management for Large Language Model Serving with PagedAttention (vLLM)`, SOSP 2023 / arXiv:2309.06180。成熟度：已落地服务系统。
