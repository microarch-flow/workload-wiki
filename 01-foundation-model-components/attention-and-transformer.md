# Attention and Transformer

上级：[Foundation Model Components](README.md)
相关：[Attention Variants and Efficiency](attention-variants-and-efficiency.md), [Transformer Workload](../06-chip-workload-analysis/transformer-workload.md)

## 这页在回答什么问题

这页回答 Attention 和 Transformer 为什么成为现代视觉、语言、自动驾驶、VLA、World Model 的通用结构。重点不是推导 attention 公式，而是理解它的计算结构、数据流，以及为什么它会把 workload 从规整局部计算推向 token 交互、矩阵乘、cache 和 layout 管理。

## Attention 的计算结构

Attention 是内容相关的信息读取机制。给定 query、key、value，模型先用 query 和 key 计算相似度，再用 softmax 得到权重，最后对 value 加权求和。

```text
input tokens
   ->
linear projection: Q, K, V
   ->
Q x K^T score matrix
   ->
softmax
   ->
score x V aggregation
   ->
output tokens
```

这个结构的关键变化是：读取位置不再由固定卷积核决定，而由输入内容决定。因此 attention 适合建模长距离依赖、多对象关系、多模态对齐和 query-based 输出。

Self-Attention 中 Q/K/V 来自同一组 token，适合让图像 patch、语言 token、BEV token、latent token 彼此交互。Cross-Attention 中 query 和 KV 来自不同来源，适合用 object query、map query、planning query、action query 从视觉、BEV 或语言上下文中读取信息。

常见误解：Attention 只是“更大的卷积”。实际上，卷积的访问模式由 kernel 和 feature map 坐标固定，attention 的访问模式由 token 内容和 score 决定；这会显著改变访存、并行和 cache 行为。

## Transformer Block 的数据流

Transformer 把 attention 组织成可堆叠模块。一个典型 block 包含 norm、multi-head attention、residual、MLP/FFN。

```text
tokens
   ->
norm
   ->
multi-head attention
   ->
residual add
   ->
norm
   ->
MLP / FFN
   ->
residual add
```

Attention 负责 token 间交互，MLP/FFN 负责每个 token 的通道变换，residual 和 norm 保证深层网络稳定。现代模型多采用 Pre-Norm，因为深层堆叠更稳定；RMSNorm 则在部分大模型中替代 LayerNorm，以降低归一化开销。

从 workload 角度看，Transformer block 不是只有 attention。QKV projection、output projection 和 FFN 都是大 GEMM；LayerNorm/RMSNorm 的 FLOPs 不高，但会读写整段 activation；reshape/transpose 会改变 layout，可能引入额外搬运。

典型量级上，视觉或多模态 encoder 常见 token 数可以从几百到几千不等；LLM/VLM 的文本上下文可以从几千 token 到十万 token 级别。token 数每增加一倍，full attention 的 score matrix 规模约增加四倍，因此 workload 分析必须把 `sequence length` 写成显式参数。

## 为什么它在自动驾驶、机器人和 World Model 中重要

自动驾驶需要多对象、多视角、多时刻的信息交互。Transformer 的 query 机制让 object query、map query、track query、planning query 可以从共享场景表示中读取不同任务所需的信息。

机器人 VLA 需要融合 vision token、language token 和 action token。Transformer 提供统一 token 交互框架，所以可以把观察、任务条件和动作输出放进同一套模型里。

World Model 需要建模 latent state、历史上下文和 action-conditioned future。Transformer 的 temporal attention 和 cross-attention 适合表达这种“当前状态 + 动作条件 -> 未来状态”的关系，但长序列和 rollout 会放大 cache、带宽和延迟压力。

## 演进路径

Transformer 的核心演进可以理解为四步。

| 阶段 | 设计动机 | Workload 变化 |
| --- | --- | --- |
| RNN / Conv -> Self-Attention | 解决长距离依赖和全局关系建模 | 从递推或局部卷积转向 QKV、score matrix、softmax 和聚合 |
| Single-head -> Multi-head | 让不同 head 学不同关系子空间 | 增加 head 维度并行，但 concat 和 projection 仍需同步 |
| Encoder-only / Decoder-only / Encoder-decoder | 匹配理解、生成和条件生成任务 | 推理形态从 full sequence 到 autoregressive decode 分化 |
| 单模态 Transformer -> 多模态 Transformer | 统一视觉、语言、动作、状态 | cross-attention 和 token layout 管理变得更重要 |

常见误解：Transformer 就等于 LLM。实际上，Transformer 是 token 交互框架；LLM 只是它的一个应用。ViT、BEVFormer、UniAD、OpenVLA、World Model 都可能使用 Transformer，但它们的 workload 参数和瓶颈不同。

## 一句话理解

Attention 把信息读取变成内容相关的 token 交互，Transformer 把这种交互组织成可堆叠的通用架构；它的硬件代价集中在 GEMM、attention score/value 聚合、norm、layout 变换和状态缓存。

## Workload Characterization

- 计算密度：prefill 或 encoder 阶段的 QKV、FFN 是高计算密度 GEMM，通常偏 compute-bound；decode 或短序列小 batch 阶段因为 KV cache 和权重读取占比上升，容易转为 memory-bandwidth-bound。
- 访存模式：GEMM 访问相对规整；attention score、softmax、value aggregation 会产生大中间张量；transpose、reshape、head split/merge 会破坏连续 layout；decode 阶段存在 stateful KV cache 访问。
- 并行性：prefill 可沿 batch、token、head、hidden dimension 并行；autoregressive decode 受 token 依赖限制，只能在 head、batch 和模型内部并行。
- 数据复用：weight 在 GEMM 中可复用，Q/K/V block 可通过 tiling 复用；KV cache 是 decode 的主要状态复用对象，但也带来容量和带宽压力。
- 量化敏感度：GEMM/FFN 通常适合 INT8/FP8；softmax、norm、attention score、KV cache 低比特需要谨慎处理数值范围和误差累积。
- 瓶颈类型：长序列 encoder 的瓶颈通常是 compute + activation memory；decode 的瓶颈通常是 bandwidth + capacity；端侧 VLA/AD 小 batch 场景还会受 latency 和 layout overhead 限制。
- 对硬件的核心需求：需要高利用率 GEMM、可融合 norm/activation、attention tiling、KV/state cache 管理，以及高效 layout transform。

## 参考来源

- Vaswani et al., `Attention Is All You Need`, NeurIPS 2017, arXiv:1706.03762。
- Dosovitskiy et al., `An Image is Worth 16x16 Words: Transformers for Image Recognition at Scale`, ICLR 2021, arXiv:2010.11929。
