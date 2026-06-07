# Attention and Transformer

上级：[Foundation Model Components](README.md)
相关：[Attention Variants and Efficiency](attention-variants-and-efficiency.md), [Transformer Workload](../06-chip-workload-analysis/transformer-workload.md)

## 这页在回答什么问题

这页要把 Transformer block 当成一个可计量的计算对象解剖开：每个算子的矩阵形状、FLOPs 量级、产生多大的中间张量，以及多头、softmax、norm、residual 各自的机制为什么是现在这样。推理阶段（prefill vs decode）的工作点差异不在这里展开，那是 [Transformer Workload](../06-chip-workload-analysis/transformer-workload.md) 的事；这页负责把"一个 block 到底在算什么、代价压在哪"讲到能写进架构模型的程度。

## 为什么它有效：直觉与类比

在啃形状和 FLOPs 之前，先建立一个能贯穿全篇的直觉：attention 本质是一次**按内容检索的图书馆查询**。你带着一个问题（query）走进图书馆，每本书的书脊上有一个标签（key），你不是按书架位置取书，而是把问题和每个标签比对相似度，相似度经 softmax 变成一份"注意力预算"——大部分预算押在最相关的几本书上，再按这个预算把书的内容（value）混合成答案。对应到机制：`Q·Kᵀ` 是"问题对所有标签打分"，softmax 是"把分数变成总和为 1 的预算"，`A·V` 是"按预算混合内容"。

这个类比直接解释了它为什么比卷积有效。卷积像"只能问固定座位的邻座"——访问的对象由坐标写死，要建立相隔很远两个 token 的关系，得堆很多层让感受野慢慢爬过去。attention 是"全场按相关性点名"，一步就能让第 1 个 token 直接读第 1000 个 token，只要内容相关。代价也由此而来：全场点名要给每对 token 都打一次分，于是有了 `N²` 的分数矩阵——它的能力和它的开销是同一个机制的两面。

多头可以类比成**一组各有专长的审稿人同时看同一份稿子**：一个盯语法一致性、一个盯前后指代、一个盯事实，各自给出注意力分布，最后汇总。单个审稿人（单头）容易把意见平均掉，多个审稿人能在不同维度上同时保持尖锐——这就是多头在不增加 FLOPs 的前提下提升表达力的来源。

residual 则像**给原文挂一条直通车，每层只在原稿上贴增量批注，而不是重抄一遍全文**。因为原文始终有一条不被改写的通路，梯度能顺着这条通路直达浅层，模型才敢堆到几十上百层而不退化——这也是下文 Pre-Norm 要保护的那条 identity 通路。

这些类比不是用来替代后面的形状分析，而是给它一个落点：当你在 Workload 小节看到"分数矩阵 134MB 要往返 HBM"时，记得那 134MB 就是"给全场每对人都打一次分"的物理代价。

## Attention 的计算结构与真实形状

Attention 是内容相关的信息读取：用 query 和 key 算相似度，softmax 成权重，再对 value 加权求和。读取位置不再由卷积核坐标固定，而由输入内容决定——这一句是它和卷积的本质分界，也是它访存、并行、cache 行为全部改变的根源。

把形状写出来才能算代价。设一个 block 的 hidden size 为 `d`，输入 `N` 个 token，单 head 情形下：

```text
X: [N, d]
Q = X·W_Q, K = X·W_K, V = X·W_V        # 三个 [d, d] 投影
S = Q·Kᵀ / sqrt(d_k)                    # [N, d]·[d, N] -> [N, N] 分数矩阵
A = softmax(S, axis=-1)                 # [N, N]，逐行归一化
O = A·V                                 # [N, N]·[N, d] -> [N, d]
Y = O·W_O                               # output projection [d, d]
```

两个量级要记住。其一，三个投影加 output projection 是规则大 GEMM，计算量随 `N·d²` 线性增长。其二，`Q·Kᵀ` 和 `A·V` 两步各是 `N²·d` 量级，随序列长度**平方**增长——这就是 attention 的 O(N²) 来源，也是长序列一切麻烦的根。token 数翻倍，投影部分翻倍，但分数矩阵规模翻四倍。

谁主导取决于 `N` 和 `d` 的相对大小。把单 token 的 MAC 数摊出来：线性部分（QKV + output + FFN）约 `12d²`（QKV 是 `3d²`、output `d²`、FFN 按 4 倍扩展是 `8d²`），attention 部分约 `2N·d`。两者相等的临界点是 `N ≈ 6d`。以 `d=1024` 为例，`N` 小于约 6000 时 FFN 和投影主导 FLOPs，attention 的平方项还不是大头；`N=16384` 这种长上下文下，attention 平方项才反超成为主成本。这个临界点决定了"这个 workload 是 GEMM-bound 还是 attention-bound"，是架构判断的第一刀。

常见误解：attention 只是更大的卷积。实际上卷积的访问模式由 kernel 和 feature map 坐标静态固定，编译期就能确定数据搬运；attention 的访问由 token 内容和分数动态决定，分数矩阵 `[N,N]` 是运行时才产生的大中间量，这让它的访存和 cache 行为和卷积完全不是一回事。

## 为什么 softmax 前要除 sqrt(d_k)

`Q·Kᵀ` 的每个元素是两个 `d_k` 维向量的点积。若各分量近似零均值单位方差，点积的方差正比于 `d_k`，于是 `d_k` 越大、logits 的幅度越大。softmax 在大 logits 上会饱和成接近 one-hot，梯度趋零、训练不稳。除以 `sqrt(d_k)` 把 logits 方差拉回 1 量级，是为了让 softmax 工作在梯度健康的区间。这不是经验性的缩放，而是有方差分析支撑的——记住这点，因为它解释了为什么 KV cache 低比特时 attention 分数对数值范围比 GEMM 更敏感。

## 多头：为什么不改 FLOPs 却要拆

多头把 `d` 维拆成 `h` 个 `d/h` 维子空间，每个 head 独立做一遍 attention，最后 concat 再过 `W_O`。关键事实：`h` 个 head 各算 `N²·(d/h)`，求和恰好还是 `N²·d`——多头**不增加** attention 的 FLOPs，也不增加投影的 FLOPs（`W_Q/W_K/W_V` 仍是 `[d,d]`，只是按 head 切块）。

那拆它图什么？图的是表达力和并行结构。单个 softmax attention 本质是对 value 做一次凸组合，倾向于"平均";多个 head 让模型在不同子空间里同时建立不同类型的关系（一个 head 盯位置邻近、另一个盯语义共指），再融合。对硬件而言，多头给出一条天然的并行轴（head 之间无依赖，可分到不同 tile），代价是 concat 和 `W_O` 需要一次跨 head 的同步点。

这也解释了 GQA/MQA 这类变体的动机：decode 时 KV cache 的容量和带宽正比于 KV 的 head 数，让多个 query head 共享一组 KV head，就能在几乎不损精度的前提下把 KV cache 压到 1/4 甚至更小。机制细节见 [Attention Variants and Efficiency](attention-variants-and-efficiency.md)。

## Transformer Block：逐算子的代价分布

一个 Pre-Norm block 的数据流是 `x -> norm -> attention -> +x -> norm -> FFN -> +x`。从计算量看，block 里 FLOPs 最大的不是 attention 而是 FFN：FFN 两个线性层按 4 倍扩展是 `8d²/token`，比 QKV+output 的 `4d²` 还大一倍。所以"Transformer 慢"在中短序列下首先是 GEMM 的事，不是 softmax 的事。

但 FLOPs 不等于瓶颈。真正压内存的是 attention 的中间张量：分数矩阵 `A` 的形状是 `[h, N, N]`。取 `h=16`、`N=2048`、FP16，单层就是 `16 × 2048² × 2B ≈ 134 MB` 的中间 activation，而且要写出再读回做 softmax 和 `A·V`。这块 activation 的反复进出 HBM，正是 [FlashAttention](attention-variants-and-efficiency.md) 要用 tiling + online softmax 消灭的对象——它不减 FLOPs，减的是这 134 MB 的物化和搬运。这是"FLOPs 小但 memory-bound"的典型例子，也是为什么不能只看算力估 attention。

norm、residual、reshape 这些算子 FLOPs 可以忽略，但不能在建模时忽略：RMSNorm/LayerNorm 要把整段 `[N,d]` activation 读一遍做归约再写回，是纯访存算子;head 的 split/merge 是 transpose，会打断连续 layout、引入额外搬运；residual 是 elementwise add，但它强制 attention 输出和输入同精度对齐。在 decode 这种 latency 敏感、GEMM 退化成瘦长矩阵的阶段，这些"小算子"的固定开销占比会被放大（见 06）。

## 为什么 Pre-Norm、为什么 RMSNorm

早期 Transformer 是 Post-Norm（`norm(x + sublayer(x))`），深层堆叠时残差路径上挂着 norm，初始化阶段梯度幅度沿深度漂移，需要 learning-rate warmup 才能稳。Pre-Norm（`x + sublayer(norm(x))`）把 norm 挪进残差分支内部，留出一条干净的 identity 通路，梯度能直达浅层，所以现代深层大模型几乎都用 Pre-Norm——这是"因为深层稳定性，所以改结构"的直接案例。

RMSNorm 在 Pre-Norm 基础上进一步省：它去掉减均值和 bias，只用 `x / RMS(x) · g` 缩放。少了一次求均值的归约和一组偏置参数，访存和算子数都降，质量基本持平。对一个本就被 norm 这类访存算子拖慢 decode 的 workload，这种省是直接的延迟收益。

## 它为什么成为自动驾驶、机器人、World Model 的通用底座

这些场景的共同需求是"用一个 query 从共享场景表示里读出任务所需信息"。自动驾驶里 object query / map query / track query / planning query 从同一份 BEV 或多视角 token 里各取所需（UniAD 类）；机器人 VLA 把 vision / language / action token 放进同一套 attention 做对齐；World Model 用 temporal/cross attention 表达"当前 latent state + action 条件 -> 未来 state"。Transformer 提供的就是这套统一的 token 交互框架——但代价是长序列和 rollout 会把上面分析的 `N²` 中间张量、KV cache 容量和带宽压力一并放大。

常见误解：Transformer 就等于 LLM。实际上 Transformer 是 token 交互框架，LLM 只是它的一个应用；ViT、BEVFormer、UniAD、OpenVLA、World Model 都用它，但它们的 `N`、`d`、prefill/decode 比例、KV cache 规模差异巨大，workload 画像因此完全不同。

## 一句话理解

Transformer block 的代价分两条线：随 `N·d²` 走的规则 GEMM（FFN 最大），和随 `N²` 走的 attention 中间张量（长序列下反超、且 memory-bound）；`N≈6d` 是两条线的交点，也是判断这个 workload 性格的第一个开关。

## 演进与未来方向（判断）

下面的预测都标注为判断，锚定的是 2024-2026 可观察到的真实趋势和已发表工作，不是确定的路线图；对架构师而言关键不是"哪个模型会赢"，而是"主流方向会把 workload 推向哪里"。

已经发生的演进有一条清晰主线：`N²` 是结构性的痛，整个领域都在想办法绕开它，但分成两条路。一条是**换掉 attention 的核**，用线性或次平方的序列算子替代全连接打分——Mamba/SSM（arXiv:2312.00752）的状态空间递推、各类 linear attention 都属此类，详见 [Mamba and SSM](mamba-and-ssm.md)。另一条是**保留 full attention 但用系统手段摊薄代价**——FlashAttention 消物化、GQA/MQA 压 KV head、PagedAttention 管 KV cache 分页。

我的判断是，短期内纯线性模型不会全面取代 attention，**hybrid 架构会成为主流折中**：大部分层用 SSM/线性算子扛长序列的吞吐，少数关键层保留 full attention 做精确的长程检索（Jamba、各类 attention-SSM 混合已是这个形态）。理由是纯线性模型的固定大小状态在"从长上下文里精确召回某个具体事实"这类任务上仍弱，而这正是 full attention 的强项——能力来自它显式保留了全部 pairwise 关系。

第二条判断是关于 KV cache：它正从"一个可选的推理优化"变成"架构一等公民"。MLA（multi-head latent attention，把 KV 压到低维 latent 再缓存）、cross-layer KV sharing 这类把 cache 结构性压小的设计会越来越靠近模型本体设计，而不是事后补丁。驱动力很直接——decode 的瓶颈是 KV cache 的容量和带宽（见 06），谁能把 cache 压一个数量级，谁就能在同样的 HBM 上服务更长上下文或更大 batch。

这两条对 workload 和硬件的含义是实在的，不是清谈。如果 hybrid 成主流，NPU 就不能只为 dense GEMM + full attention 这两种算子优化，还必须高效支持 SSM 的状态递推（顺序扫描、状态读改写）——这是一类访存模式和并行结构都和 GEMM 不同的新算子，会直接改写 archax 的 Capability 轴，并在 Interaction 轴上引入"状态递推"这种既非纯计算也非纯搬运的新交互。换句话说，为今天的 Transformer 设计的加速器，可能在 2-3 年内面对一个核心算子已经改变的 workload。这正是这份 wiki 要提前预判的事。

## Workload Characterization

以一个 `d=1024`、`h=16`、FFN 4 倍扩展的 encoder block 为基准刻画（推理阶段 prefill/decode 的分裂见 [Transformer Workload](../06-chip-workload-analysis/transformer-workload.md)，此处只刻画 block 本身的结构性特征）。

计算密度：FFN 和 QKV/output projection 是高 arithmetic intensity 的大 GEMM，单 token 约 `12d² ≈ 12.6M` MAC，权重充分复用时 compute-bound；attention 的 `Q·Kᵀ`、`A·V` 是 `2N·d/token`，本身 intensity 不高，且夹着 softmax 这种低强度归约，长序列下成为 memory-bound 的主因。临界点 `N≈6d`：短序列 GEMM 主导，长序列 attention 平方项主导。

访存模式：投影和 FFN 是连续大 GEMM，对 DRAM row locality 友好；分数矩阵 `[h,N,N]`（本例 `N=2048` 时单层约 134 MB）是最大中间 activation，朴素实现要物化并往返 HBM；head split/merge 的 transpose 打断连续 layout；softmax 需逐行归约。

并行性：head 之间完全无依赖，是天然并行轴；token 维在 encoder/prefill 可并行，在 autoregressive decode 受因果依赖限制只能逐步推进；GEMM 内部 `N`、`d` 两维都可切分。

数据复用：投影/FFN 权重在 batch 和 token 维上高度复用，是 tiling 和片上 buffer 的主要收益对象；attention 内 Q/K/V block 可通过 tiling 复用（FlashAttention 的核心）；多头共享同一份输入 activation。

量化敏感度：FFN、QKV/output projection 通常能 INT8/FP8；attention 分数因为前述 `sqrt(d_k)` 缩放后的数值范围、以及 softmax 的指数放大，对低比特更敏感；norm 的归约通常保高精度。

瓶颈类型：中短序列 encoder 偏 compute + activation-memory（那块 `[h,N,N]` 中间量）；长序列被 attention 平方项推向 bandwidth；逐算子看，transpose/norm/softmax 这些非 GEMM 算子在 latency 敏感场景会成为隐性瓶颈。

对硬件的核心需求：高利用率的大 GEMM 通路（FFN/projection），能避免物化分数矩阵的 attention tiling 支持（决定长序列是否可行），可融合的 norm/softmax/elementwise 通路（减小算子间往返），以及高效的 layout/transpose 支持。这些需求在 06 章会落到 RAM 的 row locality、DMA 的 double buffering、NOC 的跨 head reduction 上——在 archax 抽象里，attention 的中间张量物化与否是 Interaction 轴上一个必须显式建模的开关，而非可平均掉的细节。

## 参考来源

- Vaswani et al., `Attention Is All You Need`, NeurIPS 2017, arXiv:1706.03762。成熟度：已落地基础架构（缩放点积、多头、sqrt(d_k) 缩放的原始出处）。
- Dosovitskiy et al., `An Image is Worth 16x16 Words: Transformers for Image Recognition at Scale`, ICLR 2021, arXiv:2010.11929。成熟度：已落地，ViT 把同一 block 用于视觉。
- Xiong et al., `On Layer Normalization in the Transformer Architecture`, ICML 2020, arXiv:2002.04745。成熟度：已落地，Pre-Norm vs Post-Norm 稳定性分析的代表工作。
- Zhang & Sennrich, `Root Mean Square Layer Normalization`, NeurIPS 2019, arXiv:1910.07467。成熟度：已落地，RMSNorm 出处。
