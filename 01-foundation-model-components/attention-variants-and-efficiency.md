# Attention Variants and Efficiency

上级：[Foundation Model Components](README.md)
相关：[Attention and Transformer](attention-and-transformer.md), [Transformer Workload](../06-chip-workload-analysis/transformer-workload.md), [BEV Workload](../06-chip-workload-analysis/bev-workload.md)

## 这页在回答什么问题

[Attention and Transformer](attention-and-transformer.md) 已经说清 full attention 的代价压在 `N²` 的分数矩阵上。这页接着回答：当 `N` 因为高分辨率、多视角、长视频、多模态而爆炸时，业界用哪几类机制把这个代价压下去，每一类**究竟在省什么**（FLOPs、HBM 往返、还是 KV cache 容量），以及哪一类对 deterministic NPU 友好、哪一类反而把规则计算换成了难调度的不规则访存。核心是看懂机制差异，不是记模型名。

## 为什么它有效：直觉与类比

把 full attention 想成"开全员大会，每个人发言前都要和在场所有人逐一握手打分"。人数 `N` 翻倍，握手次数翻四倍——会议室再大也开不动。各种高效 attention 本质是四种"少握手"的开会方式，看懂它们就看懂了整条优化线。

FlashAttention 是**换一种记录方式而不减握手**：握手照旧，但不再把整张 `N×N` 的握手记录表抄到大白板（HBM）上再读回来，而是在手边的小本子（SRAM）上一边握一边滚动更新汇总，握完就只留汇总。它一次握手都没省，省的是"把巨大中间表搬进搬出"的功夫——这对应到机制是 online softmax：维护一个滚动最大值和滚动归一化和，每来一块 K/V 就增量修正已累积的输出，全程不物化 `[N,N]`。

Window attention 是**改成分组小会**：不再全员握手，只在每个小组内部握手，组与组之间靠每轮换一批人（shifted window）慢慢串信息。Sparse attention 是**只和名单上的人握手**：每个人有一份预定义的联系人清单（局部邻居 + 几个全局代表）。Deformable attention 最激进，是**只和自己临时点名的几个人握手**：每个 query 根据内容预测几个采样点，只读那几个点——握手次数从 `N` 降到常数，但"临时点名"意味着事先不知道要读谁，访问位置是动态的。

这四种方式的代价排序很说明问题：FlashAttention 不改结果、访问规则，对硬件最友好；window/block-sparse 把大矩阵换成规则小矩阵，尚可映射；deformable 把规则矩阵乘换成动态 gather——FLOPs 最省，但对 NPU 最难，因为它把"算得多"换成了"访存乱"。这条"省算力往往是拿访存规则性去换"的规律，是这页最值得记住的判断。

## FlashAttention：把 attention 重新定性为 IO 问题

回到 [Attention and Transformer](attention-and-transformer.md) 里那个数字：`h=16, N=2048, FP16` 时分数矩阵单层约 134 MB。朴素实现要把这 134 MB 写回 HBM 做 softmax，再读回来算 `A·V`。问题是 attention 的算术强度本就不高，瓶颈是这几百 MB 在 HBM 和计算单元之间的往返带宽，而不是 MAC。

FlashAttention 的解法是把 Q/K/V 切成能放进 SRAM 的 tile，逐 K/V tile 流式计算局部 score 和局部 softmax，用 online softmax 维护一个滚动最大值 `m` 和滚动归一化和 `l`，每处理一个新 tile 就按新的 `m` 重新缩放已经累加的输出。这样全程只在 SRAM 里周转，`[N,N]` 矩阵从不落 HBM。它**不改变 exact attention 的数学结果，也不减 FLOPs**，减的是 HBM 访问量——从随 `N²` 物化中间矩阵，降到只需把 Q/K/V 各读一遍量级。

对架构师，这一篇的启发不是"又一个 attention kernel"，而是一条方法论：attention 必须和 memory hierarchy 一起设计，单看 TOPS 估不出它的性能。FlashAttention-2、FlashAttention-3 进一步榨取 GPU 的异步拷贝和 FP8 通路，但内核思想不变——online softmax 让 attention 可以 streaming，这本身就该是 NPU 的一个一等原语，而不是拼凑出来的 kernel。

## Window / Local Attention：用局部性换平方

Window attention 只在固定窗口内做 attention，[Swin Transformer](vision-transformer-backbone.md) 是视觉代表。取窗口 `7×7=49` 个 token：每个窗口的 attention 是 `49²` 量级，窗口数是 `N/49`，总代价从 `O(N²)` 降到 `O(N·49)` 这种对 `N` 线性的量级。以 `N=4096`（64×64 patch 网格）为例，full attention 是约 1678 万对 token，换成 7×7 窗口后约 85 个窗口各 `49²`，总量约 20 万——差约 80 倍。shifted window 让信息每层换一批窗口边界逐步跨窗传播，弥补"只看局部"的损失。

代价不在 FLOPs 而在 layout：window partition、shift、reverse、relative position bias 都是 token 重排和索引操作。端侧硬件若不能高效做这些重排，理论上省下的 FLOPs 不会变成实际延迟下降——这又是一次"省算力换访存复杂度"的具体落点。

## Sparse / Block-sparse 与 GQA/MQA：两种不同的"稀疏"

Sparse attention 让每个 token 只连一部分 token（局部 + 全局代表 + 稀疏块）。关键区分是**规则 block-sparse 比随机 sparse 对 NPU 友好得多**：block 仍是矩阵块，能喂进 MAC 阵列；细粒度动态稀疏理论省算力，但 metadata、索引、负载不均、cache miss 经常把收益吃光。NPU 能高效支持的稀疏，是块结构固定、编译期可预测的那种。

GQA/MQA 稀疏的是另一个维度——不是 token 连接，而是 KV head。decode 的 KV cache 容量和带宽正比于 KV 的 head 数（见 [Transformer Workload](../06-chip-workload-analysis/transformer-workload.md)），让多个 query head 共享一组 KV head 就能直接压 KV cache。Llama-2 70B 用 GQA 把 64 个 query head 配 8 个 KV head，KV cache 缩到 1/8，精度几乎无损。这是"结构性减 cache"的典型，和减 FLOPs 是两回事——它针对的是 decode 的带宽与容量瓶颈，不是 prefill 的算力。

## Cross / Deformable / Temporal：不对称与动态访存

Cross-attention 的 query 和 KV 来自不同来源（自动驾驶的 object/map/planning query，VLA 的 action query，World Model 的 action-conditioned latent query）。它的 workload 特征是 `Q_len` 和 `KV_len` 严重不对称：query 可能只有几十上百，KV 来自高分辨率视觉或多帧历史，可达上万。瓶颈因此在 KV 的读取与重用，优化方向是让 KV cache、feature cache 的 layout 适合被少量 query 反复读，而不是减 query。

Deformable attention（Deformable DETR）让每个 query 只在每个尺度采样固定 `K` 个点（典型 `K=4`），代价从 `O(N·N)` 降到 `O(N·K)`。但采样点由预测的 offset 决定，位置不连续、运行时才知道，这破坏 DRAM row locality 和 DMA burst。对 deterministic NPU，它需要专门的 gather/scatter、index load、双线性插值和缓存策略——是"FLOPs 大降但访存大乱"的最极端例子，详见 [BEV Workload](../06-chip-workload-analysis/bev-workload.md)，BEVFormer 正是靠它做 view transform。

Temporal/causal attention 的核心参数不是单帧 token 数，而是 `spatial tokens × history length`；若是 autoregressive decode 还要维护 KV cache。长历史提升表征但把容量和带宽压力转给 state cache。

## 一句话理解

高效 attention 不是"把 attention 变小"，而是按任务结构选一种"少握手"的方式——FlashAttention 省 HBM 往返且不改结果（对硬件最友好），window/block-sparse 省平方项（尚可映射），GQA/MQA 省 KV cache（治 decode 带宽），deformable 省 FLOPs 但换来动态 gather（对 NPU 最难）。

## 演进与未来方向（判断）

以下为判断，锚定 2024-2026 真实工作。两条趋势对架构师直接相关。

第一，**exact 与 approximate 两条路在收敛成"硬件感知的 attention"**。早期是二分法：要么 IO-aware 但仍 full（FlashAttention），要么近似但省算力（sparse/linear）。现在的方向是合流——既要 IO-aware 的 streaming，又要结构化稀疏，还要低比特（FlashAttention-3 已用 FP8）。我的判断是，未来的高效 attention 会越来越像"为特定 memory hierarchy 编译出来的 streaming sparse kernel"，而非某个固定算法。对 NPU 的含义很实在：online softmax、streaming tiling、block-sparse 的块结构支持、gather/scatter，应当作为一等 datapath 原语进 Capability 轴，而不是靠通用算子拼。

第二，**KV cache 压缩从推理 trick 上升为模型结构**。GQA 已是标配，MLA（multi-head latent attention，把 KV 投影到低维 latent 再缓存）这类把 cache 结构性压一个数量级的设计正在并入模型本体。判断是 decode 主导的部署里，"KV 的有效比特数"会成为和参数量同等重要的一级架构指标——谁的 KV cache 小，谁就能在同样 HBM 上服务更长上下文或更大 batch。这条直接喂给 06 的端侧/云端芯片需求分析。

风险提示：deformable、动态 sparse 这类"理论省、调度难"的算子，是否值得在 NPU 上做专用支持，取决于目标 workload（BEV/检测重度依赖 deformable，纯 LLM 则不需要）——这是个该在 archax 里按目标 workload 显式权衡的 Capability 取舍，不是普适结论。

## Workload Characterization

计算密度：FlashAttention 不减 FLOPs 但通过砍 HBM IO 提升有效算术强度（把 memory-bound 的 attention 拉回算力可用区）；window/block-sparse 把 `N²` 降到近线性；deformable 把 FLOPs 降到 `O(N·K)` 但有效算力利用率低（动态访存填不满阵列）。

访存模式：FlashAttention 是规则 SRAM tiling，HBM 访问从 `O(N²)` 量级降到读 Q/K/V 各一遍量级；window attention 需 partition/shift/reverse 的 token 重排；block-sparse 依赖块元数据但块内规则；deformable 是动态 gather/scatter，破坏 row locality；cross-attention 重点是不对称 KV 的重复读取。

并行性：head、window、block 天然并行；causal decode 受时间依赖串行；deformable/sparse 的负载均衡取决于索引分布，最坏情况严重不均。

数据复用：FlashAttention 复用 SRAM tile 内的 Q/K/V；GQA/MQA 让多 query head 复用同一组 KV（直接减 cache）；cross/temporal attention 复用 KV/history cache；window attention 复用窗口内 token。

量化敏感度：QKV/FFN projection 易 INT8/FP8；softmax、`sqrt(d_k)` 缩放后的 score、deformable 的插值与动态 offset 对数值更敏感，低比特需验证。

瓶颈类型：full attention 是 compute + activation-memory；FlashAttention 把瓶颈从 IO 解放、回到算力；deformable/动态 sparse 容易 irregular-access-bound；causal decode 容易 KV-bandwidth-bound。

对硬件的核心需求：online-softmax/streaming tiling 原语、block-sparse 块结构支持、高效 gather/scatter + 插值、layout/transpose 优化、KV cache 的紧凑布局与共享——这些需求在 06 落到 RAM 的 row locality、DMA 的 burst/gather、NOC 的跨 head reduction；在 archax 里，"是否物化分数矩阵""是否支持动态采样"是 Capability 与 Interaction 轴上必须显式建模的开关。

## 参考来源

- Dao et al., `FlashAttention: Fast and Memory-Efficient Exact Attention with IO-Awareness`, NeurIPS 2022, arXiv:2205.14135。成熟度：已落地，IO-aware exact attention 的奠基工作。
- Dao, `FlashAttention-2: Faster Attention with Better Parallelism and Work Partitioning`, arXiv:2307.08691。成熟度：已落地。
- Liu et al., `Swin Transformer: Hierarchical Vision Transformer using Shifted Windows`, ICCV 2021, arXiv:2103.14030。成熟度：已落地，window/shifted-window 代表。
- Ainslie et al., `GQA: Training Generalized Multi-Query Transformer Models from Multi-Head Checkpoints`, EMNLP 2023, arXiv:2305.13245。成熟度：已落地，GQA 出处。
- Zhu et al., `Deformable DETR: Deformable Transformers for End-to-End Object Detection`, ICLR 2021, arXiv:2010.04159。成熟度：已落地，deformable attention 代表。
- DeepSeek-AI, `DeepSeek-V2: A Strong, Economical, and Efficient Mixture-of-Experts Language Model`, arXiv:2405.04434。成熟度：已落地，MLA（multi-head latent attention）出处。
