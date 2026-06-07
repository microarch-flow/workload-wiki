# Vision Transformer Backbone

上级：[Foundation Model Components](README.md)
相关：[CNN Backbone](cnn-backbone.md), [Attention and Transformer](attention-and-transformer.md), [Transformer Workload](../06-chip-workload-analysis/transformer-workload.md)

## 这页在回答什么问题

这页回答 ViT 把图像从 feature map 改成 token 序列后，workload 形态发生了什么质变：token 数怎么随分辨率爆炸、它和 CNN 的代价分别压在哪、为什么高分辨率视觉必须用分层或 window 结构，以及这对端侧多模态系统意味着什么。

## 为什么它有效：直觉与类比

ViT 的直觉是**把一张图剪成一叠便利贴，然后让所有便利贴开一场全员会议**。每个 `P×P` 的 patch 拍平、投影成一个 token（便利贴），然后送进 Transformer 让它们彼此 attention。和 CNN 那个"固定印章滑动盖章、只看邻域"的直觉正好相反——ViT 一上来就允许任意两张便利贴直接交流，不管它们在画面里隔多远。

这解释了 ViT 为什么强、又为什么"难伺候"。强：建立远距离关系不用像 CNN 那样堆很多层让感受野慢慢爬，一层 attention 就能让左上角 patch 直接读右下角 patch，全局上下文来得快。难：它放弃了 CNN 自带的局部性和平移等变先验——印章模型天生知道"猫在哪都是猫"，便利贴模型不知道，得靠位置编码 + 大规模预训练把这些视觉结构从数据里学出来。所以 ViT 在小数据上常输给 CNN，在大数据 + 预训练后反超。对应到机制，这就是"弱归纳偏置换强表达力，但用数据规模来还债"。

代价的直觉也直接：全员会议的握手次数随便利贴数量平方增长（见 [Attention and Transformer](attention-and-transformer.md) 的 `N²` 分析）。而便利贴数量由分辨率决定——这是 ViT 一切 workload 问题的源头，下一节量化它。

## 从图像到 token：被平方放大的代价

输入 `H×W`、patch `P×P`，token 数约 `(H/P)×(W/P)`。224×224 配 16×16 patch 是 196 个 token；换成 1024×1024 仍用 16×16，是 4096 个 token。token 数随边长平方增长，而 full attention 的分数矩阵又随 token 数平方增长——于是**分辨率翻倍，token 数 ×4，attention 代价 ×16**。这条二阶放大链解释了为什么高分辨率视觉系统几乎都不用原始 ViT：196 token 的分数矩阵是小事，4096 token 时单层 `[h,N,N]` 中间张量（`h=16,FP16`）已是 `16×4096²×2B ≈ 537 MB`，直接压垮 activation memory 和带宽。

ViT 的设计动机正是用全局 attention 替代 CNN 的固定局部偏置——不预设邻域卷积是唯一合理的信息交互，让任意 patch 通过 attention 建关系。代价就是上面这条平方放大链，以及对预训练规模的依赖。

## ViT 和 CNN 的本质差异

CNN 的归纳偏置强（局部性、平移等变、多尺度层次天然存在），ViT 弱（依赖数据规模、预训练、位置编码学视觉结构）。落到 workload 上差异更直接：

| 维度 | CNN Backbone | ViT Backbone |
| --- | --- | --- |
| 基本表示 | `H×W×C` feature map | `N tokens × C` |
| 主算子 | Conv、BN/activation、elementwise | GEMM、attention、FFN、norm |
| 访问模式 | 规则滑窗、空间局部 | token matrix、attention 全局交互 |
| 复用 | weight/activation 空间复用强 | GEMM 复用强，但 attention 中间激活大 |
| 端侧风险 | depthwise、分支碎片化 | token 数爆炸、activation memory、norm/layout |

常见误解：ViT 一定比 CNN 先进。实际上 ViT 给出更统一的 token 表示和全局建模、更易接多模态，但端侧小模型、小数据、强实时场景里，CNN 或 CNN+Transformer 混合往往更合适——这与 [CNN Backbone](cnn-backbone.md) 末尾的判断一致。

## 分层 ViT 与 Swin：把层次结构装回来

原始 ViT 所有层 token 数不变，不天然产生 CNN 那样的多尺度 hierarchy。Swin Transformer 用 window attention + shifted window + patch merging 把层次结构装回来：window attention 把 `N²` 降到窗口内的近线性（见 [Attention Variants and Efficiency](attention-variants-and-efficiency.md)），patch merging 让深层 token 数减半、channel 加倍，形成 CNN 式的多尺度金字塔。

代价是 workload 变复杂：window partition、shift、reverse、patch merge、relative position bias 都是 token 重排和索引。对芯片，Swin 的关键不是 attention FLOPs 低，而是这些 token 重排能否被高效调度——不能高效重排的硬件，拿不到理论上的延迟收益。

## 在 AD / VLA / World Model 中的意义

自动驾驶里 ViT/Swin 可作多摄像头 image encoder，提供更适合 query-based BEV 和 E2E 的 token 表示。VLA 里 ViT 是常见 vision encoder，因为它输出的 token 容易和 language/action token 统一。World Model 里 ViT 可作 frame encoder 或 latent tokenizer 的一部分。但这些系统多数不用 full-resolution ViT：端侧常降 token 数、用分层结构、用 CNN stem 或轻量 vision encoder，避免视觉 token 直接压垮后端 VLM/VLA——视觉 token 数在多模态系统里往往是首要成本驱动，详见 [VLA Workload](../06-chip-workload-analysis/vla-workload.md)。

## 一句话理解

ViT 把视觉 workload 从规则 feature map 卷积变成 token 序列交互，换来全局建模和多模态接口，代价是 token 数随分辨率平方爆炸、attention 中间张量随之平方放大；高分辨率可行性取决于分层/window 结构能否被硬件高效调度。

## 演进与未来方向（判断）

以下为判断，锚定真实趋势。

第一，**视觉 encoder 正在把"token 数"本身当成一级设计变量来治理**。原始 ViT 固定 patch、固定 token 数；新一代往可变方向走——NaViT（arXiv:2307.06304）支持原生分辨率与变长 token packing，token merging/pruning 在推理时动态丢弃冗余 token。我的判断是，多模态时代视觉 token 数是 LLM 上下文成本的主源头，所以视觉 encoder 会越来越多地内建 token 压缩（merge/prune/分层下采样），把"喂给后端多少 token"做成可调旋钮而非固定值。对 NPU 的含义：必须高效支持可变 token 数和动态 token 合并/丢弃，而不是假设静态 shape——这给以静态调度见长的 deterministic NPU 出了真题。

第二，**自监督 ViT 表征质量的跃升会反过来固化 ViT 在多模态前端的地位**。DINOv2、加 register token 修复 artifact 等工作（见 [Contrastive Learning](contrastive-learning.md)、[JEPA and Self-supervised](jepa-and-self-supervised.md)）让 ViG 特征足够强，VLM/VLA 几乎默认用 ViT 做 vision tower。判断是端侧的折中会稳定成"CNN/轻量 stem 降分辨率 + 小 ViT 出 token"，纯 CNN 前端在需要强语义 token 的多模态任务上会逐步让位。对 archax，ViT backbone 应按 token 数显式参数化，把 196/1024/4096 token 当成不同工作点分别评估——因为它们落在 compute-bound 还是 activation-memory-bound 的位置完全不同。

## Workload Characterization

计算密度：patch embedding 和 FFN 是高算术强度 GEMM；full attention 随 token 数平方增长，中小 token 数下 GEMM 主导（compute-bound），高分辨率/多帧/多相机下 attention 的 `[h,N,N]` 中间张量（4096 token 时约 537 MB）把负载推向 activation-memory/bandwidth-bound。

访存模式：GEMM 相对规则；attention score、softmax、token reshape、window partition、patch merge 引入额外 activation 搬运与索引；分层结构的 token 数逐层变化使 shape 非恒定。

并行性：token、head、hidden、batch 可并行；端侧小 batch 靠 token/head 并行，长 token 序列受 SRAM/DRAM 容量限制。

数据复用：projection/FFN weight 可复用；attention 靠 tiling 复用 Q/K/V block（FlashAttention）；window attention 复用窗口内 token。

量化敏感度：linear/FFN 适合 INT8/FP8；LayerNorm、softmax、attention score、position bias 需谨慎，与 [Attention and Transformer](attention-and-transformer.md) 分析一致。

瓶颈类型：中小 token 数 compute-bound；高分辨率/多帧/多相机 memory-capacity/bandwidth-bound；window/shift/merge 等重排在端侧可能 latency-bound。

对硬件的核心需求：高效 GEMM、attention tiling（避免物化分数矩阵）、norm/softmax fusion、layout transform、token/window 重排，以及对可变 token 数和动态 token 合并的支持——后者是未来视觉 encoder 的关键，对静态调度的 NPU 是新挑战。

## 参考来源

- Dosovitskiy et al., `An Image is Worth 16x16 Words: Transformers for Image Recognition at Scale`, ICLR 2021, arXiv:2010.11929。成熟度：已落地，ViT 出处。
- Liu et al., `Swin Transformer: Hierarchical Vision Transformer using Shifted Windows`, ICCV 2021, arXiv:2103.14030。成熟度：已落地，分层/window ViT 代表。
- Dehghani et al., `Patch n' Pack: NaViT, a Vision Transformer for any Aspect Ratio and Resolution`, NeurIPS 2023, arXiv:2307.06304。成熟度：研究到落地之间，原生分辨率/变长 token 代表。
- Oquab et al., `DINOv2: Learning Robust Visual Features without Supervision`, arXiv:2304.07193。成熟度：已落地开源，强自监督 ViT 表征。
