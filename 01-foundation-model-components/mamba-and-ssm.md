# Mamba and SSM

上级：[Foundation Model Components](README.md)
相关：[Attention and Transformer](attention-and-transformer.md), [World Model Workload](../06-chip-workload-analysis/world-model-workload.md), [Transformer Workload](../06-chip-workload-analysis/transformer-workload.md)

## 这页在回答什么问题

这页回答 SSM 和 Mamba 为什么重新成为长序列建模的重要路线，以及它们作为 workload 与 Transformer attention 有什么不同。重点是 selective state update、scan、state cache 和 streaming inference，而不是数学形式推导。

## SSM 的基本计算结构

State Space Model 的直觉是用一个内部状态压缩历史。每个新输入到来时，模型根据当前输入和旧状态更新新状态，并输出当前表示。

```text
previous state
   + current input
   ->
state update
   ->
new state
   ->
output
```

这和 full attention 的组织方式不同。Attention 显式保留并读取大量历史 token，SSM 倾向于把历史压缩进状态。对于长序列，SSM 的吸引力在于不需要构造 `N x N` attention matrix，也不需要像 decode KV cache 那样随长度线性增长地保存所有 key/value。

常见误解：SSM 只是 RNN 换名。实际上，现代 SSM 强调结构化状态更新、并行训练、长程建模和硬件友好实现；Mamba 的 selective SSM 让状态更新对输入内容敏感，不是固定递推。

## Mamba 的 Selective Scan

Mamba 的关键是 selective state space：模型根据输入选择哪些信息写入状态、保留状态或遗忘状态。它把长序列建模从 full attention 的 token-token matrix 转成 projection、gating、selective scan 和 state update。

```text
tokens
   ->
linear projection / gating
   ->
selective scan over sequence
   ->
stateful output
```

训练时，scan 需要并行化以获得吞吐；推理时，SSM 可以以 streaming 方式维护状态。这个差异对硬件很重要：训练侧更像高吞吐 scan kernel，推理侧更像低延迟 state update loop。

Mamba-2 进一步用 structured state space duality 连接 SSM 与 attention，并改善核心 layer 的效率。对 workload 来说，这说明 SSM 路线不是“完全避开矩阵计算”，而是在 attention-like 表达和 scan/state execution 之间重新组织计算。

## 与 Transformer 的互补关系

Transformer 擅长显式 query-based 读取、多模态对齐和任意 token 交互。Mamba/SSM 擅长长序列压缩、流式状态更新和避免 quadratic attention matrix。

因此更现实的趋势不是二选一，而是混合：视觉或语言模块仍用 attention 做关键交互，长历史视频、BEV memory、trajectory history、latent dynamics 可能用 SSM 压缩。

在自动驾驶和机器人中，SSM 值得关注的场景包括 video history encoder、temporal BEV memory、planning history summarizer、robot proprioception/action history、World Model latent dynamics。2025 年前后的论文已经开始把 Mamba/SSM 用到 trajectory prediction 等长序列任务，例如 Trajectory Mamba（CVPR 2025，把 encoder-decoder 的 self-attention 改成 selective SSM，做到线性复杂度、FLOPs 降约 4 倍）；这类工作多数仍是论文阶段，应避免写成已量产方案。

典型量级上，SSM 的吸引力来自长序列：当 history length 从几十帧扩展到几百或几千 token 时，full attention 的 score matrix 按 `N^2` 增长，而 SSM 更接近按序列长度线性推进。但如果 state dimension 很大，state 读写和 scan kernel 仍会成为硬件瓶颈。

## Workload 影响

Mamba/SSM 把长序列压力从 attention matrix 和 KV cache 转移到 scan kernel、state update、门控投影和 state residency。

这带来三个直接后果：

1. 序列长度增长时，中间 attention matrix 不再按 `N^2` 放大。
2. 推理可以维护压缩状态，减少保存完整历史 token 的容量压力。
3. 硬件必须支持高效 scan/state update；如果 scan kernel 不成熟，理论复杂度优势可能无法兑现。

常见误解：Mamba 一定比 Transformer 更省。实际上，如果任务需要大量 cross-modal query 读取、精确对象关系或多路动态索引，attention 仍然更自然；如果 NPU 不支持高效 scan，Mamba 也可能被 projection、gating 和 state layout 限制。

## 一句话理解

Mamba/SSM 用状态递推压缩长历史，把 workload 从 full attention 的 `N x N` 交互转向 projection、selective scan 和 state update；它对长序列友好，但依赖硬件对 scan/stateful execution 的支持。

## Workload Characterization

- 计算密度：linear projection 和 gating 有较高计算密度；selective scan 本身依赖状态更新和序列访问，可能受 memory bandwidth、scan kernel 和并行前缀算法限制。
- 访存模式：相比 full attention 少了大 attention matrix；但需要顺序或块状访问 sequence 和 state，state layout 对 streaming inference 很关键。
- 并行性：训练可通过 scan 并行化；推理沿时间有依赖，但每步状态更新轻；batch、channel、state dimension 可并行。
- 数据复用：state 是核心复用对象；长历史不以完整 token 形式保存，而被压缩进状态。
- 量化敏感度：projection/GEMM 可低比特；state update、门控、长期状态累积需要关注数值稳定性。
- 瓶颈类型：长序列训练可能 scan-kernel-bound；端侧 streaming 推理可能 latency-bound；如果 state 大或 layout 不佳，也会 memory-bandwidth-bound。
- 对硬件的核心需求：高效 selective scan、state cache、门控投影融合、低延迟 streaming update、混合 attention/SSM pipeline 支持。

## 参考来源

- Gu and Dao, `Mamba: Linear-Time Sequence Modeling with Selective State Spaces`, arXiv:2312.00752。
- Dao and Gu, `Transformers are SSMs: Generalized Models and Efficient Algorithms Through Structured State Space Duality`, arXiv:2405.21060。
- Huang et al., `Trajectory Mamba: Efficient Attention-Mamba Forecasting Model Based on Selective SSM`, CVPR 2025 / arXiv:2503.10898。成熟度：研究阶段，Mamba 用于自动驾驶轨迹预测的代表。
