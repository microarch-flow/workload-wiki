# Attention Variants and Efficiency

上级：[Foundation Model Components](README.md)
相关：[Attention and Transformer](attention-and-transformer.md), [Transformer Workload](../06-chip-workload-analysis/transformer-workload.md), [BEV Workload](../06-chip-workload-analysis/bev-workload.md)

## 这页在回答什么问题

这页回答为什么标准 full attention 在长序列、视觉、多模态和端侧系统中不够用，以及不同 attention 变体如何改变 workload。重点是理解效率优化背后的机制，而不是记住一串模型名。

## Full Attention 的根本问题

Full attention 允许每个 token 关注所有 token，表达能力强，但代价随序列长度快速增长。对于长度为 `N`、head dimension 为 `d` 的序列，score matrix 是 `N x N`，计算和中间激活都随 `N^2` 增长。

在短序列或云端大矩阵场景中，这个代价可以被高效 GEMM 吞掉。但在视觉、多摄像头、视频、VLA 和 World Model 中，token 数来自空间、时间、模态和候选分支的乘积，full attention 很快变成 activation memory 和带宽问题。

一个具体锚点是：`N=1024` 时单个 head 的 score matrix 已经有约 100 万个元素；`N=4096` 时增加到约 1678 万个元素。即使每个元素只用 FP16，这类中间矩阵也会快速变成片外存储和带宽问题。

常见误解：优化 attention 只是减少 FLOPs。实际上，很多 attention 优化首先是在减少 HBM/DRAM 往返、attention matrix materialization、layout transform 和 cache pressure。

## FlashAttention：IO-aware 的代表

FlashAttention 的核心不是近似 attention，而是在不改变 exact attention 结果的前提下，用 tiling 和 online softmax 避免把完整 `N x N` attention matrix 写回 HBM。它把 Q/K/V 分块放入 SRAM，分块计算 score 和 softmax，并在线维护归一化结果。

```text
Q block + K/V block
   ->
SRAM tile compute
   ->
online softmax update
   ->
accumulate output
```

因为 attention 的瓶颈经常是 HBM 读写而不是单纯 MAC，所以 FlashAttention 的意义在于把 attention 从“频繁读写大中间矩阵”改成“在片上分块完成更多工作”。这对 NPU 的启发是：attention kernel 必须和 memory hierarchy 一起设计，不能只看矩阵乘算力。

## Window / Local Attention

Window attention 只在局部窗口内做 attention，Swin Transformer 是视觉领域的代表路线之一。它利用图像局部性降低 attention matrix 规模，并用 shifted window 让跨窗口信息逐层传播。

它的 workload 变化是：大规模 full attention 变成多个小窗口 attention，计算和激活下降；但窗口划分、shift、partition、reverse、token reorder 会增加 layout 操作。端侧硬件如果不能高效处理这些重排，理论 FLOPs 降低不一定转化为实际延迟下降。

## Sparse / Block-sparse Attention

Sparse attention 让 token 只连接一部分 token，可以是局部、全局 token、稀疏块或任务定义的结构。它适合长上下文和视觉/BEV 中的结构化读取。

它的代价是硬件调度复杂。规则 block-sparse 比随机 sparse 更容易映射到 NPU，因为 block 仍能形成矩阵块；细粒度动态稀疏虽然理论上省算力，但可能被 metadata、索引、负载不均和 cache miss 抵消收益。

## Cross-Attention

Cross-attention 的 query 和 KV 来自不同来源。自动驾驶中的 object/map/planning query，机器人 VLA 中的 action query，World Model 中的 action-conditioned latent query，都属于这类模式。

它的 workload 特征是 `Q_len` 和 `KV_len` 不对称。query 可能很少，但 KV 来自高分辨率视觉、多帧历史或多模态上下文，因此瓶颈经常是 KV 的读取和重用。优化方向不是单纯减少 query，而是让 KV cache、feature cache 和 cross-modal layout 更适合重复读取。

## Deformable Attention

Deformable attention 只在少量动态采样点读取特征，在检测、BEV 和多尺度视觉任务中很有吸引力。它减少了 full attention 的计算，但把问题转化为动态索引和插值采样。

从芯片角度看，它是典型的“FLOPs 下降但访存变难”。采样点来自预测 offset，访问位置不连续，可能破坏 DRAM row locality 和 DMA burst。对于 deterministic NPU，这类算子需要专门的 gather/scatter、index load、插值和缓存策略。

## Temporal / Causal Attention

Temporal attention 用于视频、自动驾驶历史窗口、机器人观察历史和 World Model rollout。Causal attention 还会引入自回归依赖。

这里的核心参数不是单帧 token 数，而是 `spatial tokens x history length`。如果是 autoregressive decode，还要维护 KV cache。长历史提升表征能力，但会把容量和带宽压力转移到 state cache。

## 一句话理解

Attention efficiency 的核心不是“把 attention 变小”，而是根据任务结构减少不必要的 token 交互、避免大中间矩阵落到外存，并让访问模式更适合硬件。

## Workload Characterization

- 计算密度：FlashAttention 通过减少 HBM/DRAM IO 提高有效计算密度；window/block attention 降低 `N^2` 规模；deformable attention 降低 FLOPs 但可能降低有效算力利用率。
- 访存模式：FlashAttention 偏规则 tiling；window attention 需要 partition/shift/reverse；sparse attention 依赖稀疏元数据；deformable attention 是动态 gather/scatter；cross-attention 重点是 KV 重用。
- 并行性：head、block、window 可并行；causal decode 受时间依赖限制；deformable/sparse 的负载均衡取决于索引分布。
- 数据复用：FlashAttention 复用 SRAM tile；cross-attention 复用 KV；temporal attention 复用 history cache；window attention 复用局部窗口内 token。
- 量化敏感度：QKV projection 和 FFN 易低比特；softmax、score scaling、插值采样和动态 offset 对数值更敏感。
- 瓶颈类型：full attention 是 compute + activation memory；FlashAttention 主要优化 IO；deformable/sparse 容易变成 irregular-access-bound；causal attention decode 容易 bandwidth-bound。
- 对硬件的核心需求：需要 attention tiling、online softmax、block-sparse 支持、gather/scatter、layout transform 优化和 state cache 管理。

## 参考来源

- Dao et al., `FlashAttention: Fast and Memory-Efficient Exact Attention with IO-Awareness`, NeurIPS 2022, arXiv:2205.14135。
- Liu et al., `Swin Transformer: Hierarchical Vision Transformer using Shifted Windows`, ICCV 2021, arXiv:2103.14030。
- Zhu et al., `Deformable DETR: Deformable Transformers for End-to-End Object Detection`, ICLR 2021, arXiv:2010.04159。
