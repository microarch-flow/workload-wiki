# Mamba and SSM

上级：[Foundation Model Components](README.md)
相关：[Attention and Transformer](attention-and-transformer.md), [World Model Workload](../06-chip-workload-analysis/world-model-workload.md), [Transformer Workload](../06-chip-workload-analysis/transformer-workload.md)

## 这页在回答什么问题

这页回答 SSM/Mamba 为什么重新成为长序列建模的重要路线，以及它作为 workload 和 attention 到底差在哪。重点是 selective state update、scan、固定大小状态、streaming inference 这几件事的机制与硬件含义，而不是状态空间的数学推导。看懂它，才能理解 [Attention and Transformer](attention-and-transformer.md) 末尾预判的 hybrid 架构会给 NPU 带来什么新算子。

## 为什么它有效：直觉与类比

SSM 的直觉是**一个边听边记笔记的人，手里只有一张固定大小的便签**。每听到一句话（一个 token），他就根据这句话更新便签上的摘要，然后接着听下一句，从不回头翻前面说过什么。听完整段，便签上的摘要就是他对全文的理解。把这张便签叫做"状态"，把"根据新输入更新便签"叫做状态递推，就是 SSM 的全部骨架。

对照 attention 的开会直觉——它是**把整本会议记录摊在桌上，每写一个新字都回头扫一遍全部历史**（这就是 `N²` 的来源）。SSM 不摊开历史，它把历史压进那张固定大小的便签里。这一个区别带来 SSM 最关键的性质：处理长度 `N` 的序列，attention 的代价随 `N²` 涨、KV cache 随 `N` 线性涨，而 SSM 的计算随 `N` 线性、**状态大小恒定不随 `N` 变**。一段 10 万 token 的上下文，attention 的 KV cache 能涨到几十 GB，SSM 的便签还是那么大——这是它对长序列的根本吸引力。

但早期 SSM/RNN 有个致命弱点：记笔记的规则是死的，不管听到什么都按同一套公式更新便签，于是重要的和废话被一视同仁地压缩，长程信息很快被冲淡。Mamba 的核心突破——**selective**——就是让记笔记的人变聪明：根据当前听到的内容，动态决定这句话值不值得重点记、哪些旧摘要该保留、哪些该遗忘。对应到机制，就是让状态更新的参数变成输入的函数，而不是固定常数。正是这个"选择性"把 SSM 的质量追到了接近 attention 的水平，让它从"RNN 的旧瓶"变成真正的竞争路线。

代价也藏在那张便签里：它就一张，固定大小。如果任务需要精确召回很久以前某个具体细节，而那个细节当时没被选中记进便签，就找不回来了。这正是纯 SSM 在"大海捞针式精确检索"上仍弱于 attention 的根本原因，也是下文 hybrid 架构存在的理由——attention 显式保留全部历史，是 SSM 的便签压缩换不来的能力。

## Selective Scan：机制与硬件的双重面孔

Mamba 把长序列建模从 attention 的 token-token 矩阵，重组成 projection、gating、selective scan、state update：

```text
tokens -> linear projection / gating -> selective scan over sequence -> stateful output
```

这里有个看似矛盾的事实：状态递推天生是顺序的（第 `t` 步依赖第 `t-1` 步的状态），怎么并行训练？答案是 selective scan 用**结合律扫描（associative scan / parallel prefix）**把顺序递推改写成可并行归约——就像求前缀和可以用树形归约 `O(log N)` 深度完成，而不必逐个累加。于是同一个算子有两副面孔：训练时是高吞吐的 parallel scan kernel，推理时是低延迟的逐步 state update loop。这个双面性对硬件很重要——训练侧要的是 scan 的吞吐，推理侧要的是单步 state 读改写的低延迟，两者优化点不同。

Mamba-2 用 structured state space duality（SSD）把 SSM 和 attention 在数学上连起来，说明 SSM 路线不是"完全避开矩阵计算"，而是在 attention-like 表达和 scan/state execution 之间重新组织计算。对架构师，这条 duality 的实际含义是：SSM 和 attention 共享一部分 GEMM 结构，但 SSM 多出"状态递推"这一类既非纯 GEMM、又非纯 attention 的新算子——而这正是现有 NPU 的盲区。

## 与 Transformer 的互补，不是替代

attention 擅长显式 query-based 读取、多模态对齐、任意 token 精确交互；Mamba/SSM 擅长长序列压缩、流式状态更新、避开 `N²` 矩阵。现实趋势不是二选一而是混合：关键交互层用 attention，长历史视频、BEV memory、trajectory history、latent dynamics 用 SSM 压缩。自动驾驶/机器人里值得用 SSM 的场景包括 video history encoder、temporal BEV memory、planning history summarizer、proprioception/action history、World Model latent dynamics。2025 年前后已有把 Mamba 用于轨迹预测的工作，如 Trajectory Mamba（CVPR 2025，把 encoder-decoder 的 self-attention 换成 selective SSM，线性复杂度、FLOPs 降约 4 倍），但多为论文阶段，不应写成量产方案。

量级上，SSM 的吸引力来自长序列：history length 从几十帧扩到几百几千 token 时，attention 按 `N²` 涨而 SSM 近线性；但若 state dimension 很大，state 读写和 scan kernel 本身仍可能成为硬件瓶颈——线性复杂度不等于无成本。

## 一句话理解

Mamba/SSM 用一张固定大小、内容自适应（selective）的"便签"压缩长历史，把 workload 从 attention 的 `N×N` 交互和线性增长 KV cache，换成 projection、selective scan 和恒定大小的 state update；它对长序列友好，但精确长程检索弱于 attention，且能否兑现线性优势取决于硬件对 scan/stateful execution 的支持。

## 演进与未来方向（判断）

以下为判断，锚定真实趋势。这条和 [Attention and Transformer](attention-and-transformer.md) 的未来方向节互为表里。

第一，**hybrid 架构（SSM + attention 混层）正在成为长上下文的主流折中**，已不只是论文设想。Jamba（AI21，arXiv:2403.19887）、NVIDIA 的 Nemotron-H/Hymba 等把多数层用 Mamba 扛吞吐、少数层保留 attention 做精确检索，在长上下文吞吐和质量间取平衡。我的判断是 2-3 年内端侧和长上下文部署会大量出现这种混合体——纯 attention 在超长上下文太贵，纯 SSM 在精确召回太弱，混合是当前能看到的稳定解。

第二条判断对架构师最关键：**hybrid 一旦成主流，NPU 的算子集合必须扩容**。今天的 NPU 为两类算子优化——dense GEMM 和 full attention。SSM 引入第三类：状态递推（selective scan）。它的访存和并行结构都和前两者不同——训练侧要高效的 associative scan（parallel prefix，访存模式像树形归约而非矩阵分块），推理侧要 state 常驻 SRAM 的低延迟读改写，且 state 的 layout 直接决定 streaming 效率。这是一类全新的 Capability，不能用 GEMM 单元凑。我的判断是：为今天纯 Transformer 设计的加速器，可能在 2-3 年内面对一个核心算子已经改变的 workload；提前在 archax 的 Capability 轴上预留"state recurrence / selective scan"这一类原语，在 Interaction 轴上把"状态递推"建模为既非纯计算也非纯搬运的第三种交互，是这份 wiki 要替架构探索提前做的功课。这与 06 的 [World Model Workload](../06-chip-workload-analysis/world-model-workload.md) 对 latent dynamics 的分析直接衔接。

风险提示：SSM 是否值得在某颗 NPU 上做专用 scan 支持，取决于目标 workload 是否真有长序列/长历史需求——纯短上下文推理用不上，长视频 World Model 则强需求。这是个该按目标 workload 显式权衡的取舍，不是普适结论。

## Workload Characterization

计算密度：linear projection 和 gating 是较高算术强度的 GEMM；selective scan 本身依赖状态更新和序列访问，可能受 memory bandwidth、scan kernel 实现、parallel-prefix 效率限制——线性复杂度不保证高利用率。

访存模式：相比 full attention 省掉大 `N×N` 矩阵；但需顺序或块状访问 sequence 与 state，state layout 对 streaming inference 至关重要；associative scan 的访存像树形归约，与 GEMM 的规则分块不同。

并行性：训练侧靠 associative scan 把顺序递推并行化（`O(log N)` 深度）；推理侧沿时间有依赖但每步状态更新轻；batch、channel、state dimension 可并行。

数据复用：state 是核心复用对象，长历史不以完整 token 形式保存而被压缩进恒定大小状态——这是相对 KV cache 的根本省法；projection weight 在序列维复用。

量化敏感度：projection/GEMM 可低比特；state update、gating、长期状态累积对数值稳定性敏感（误差会沿递推累积），需谨慎。

瓶颈类型：长序列训练可能 scan-kernel-bound；端侧 streaming 推理可能 latency-bound；若 state 大或 layout 不佳，也会 memory-bandwidth-bound。

对硬件的核心需求：高效 selective scan（parallel prefix）、state cache（常驻 SRAM）、gating/projection 融合、低延迟 streaming state update、attention/SSM 混合 pipeline 支持——其中"原生状态递推"是相对今天 NPU 算子集的关键增量，详见 [World Model Workload](../06-chip-workload-analysis/world-model-workload.md)。

## 参考来源

- Gu and Dao, `Mamba: Linear-Time Sequence Modeling with Selective State Spaces`, arXiv:2312.00752。成熟度：已落地开源，selective SSM 出处。
- Dao and Gu, `Transformers are SSMs: Generalized Models and Efficient Algorithms Through Structured State Space Duality (Mamba-2)`, ICML 2024 / arXiv:2405.21060。成熟度：已落地，SSM-attention duality。
- Lieber et al., `Jamba: A Hybrid Transformer-Mamba Language Model`, arXiv:2403.19887。成熟度：已落地开源，hybrid 架构代表。
- Huang et al., `Trajectory Mamba: Efficient Attention-Mamba Forecasting Model Based on Selective SSM`, CVPR 2025 / arXiv:2503.10898。成熟度：研究阶段，Mamba 用于自动驾驶轨迹预测。
