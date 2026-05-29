# Vision Transformer Backbone

上级：[Foundation Model Components](README.md)
相关：[CNN Backbone](cnn-backbone.md), [Attention and Transformer](attention-and-transformer.md), [Transformer Workload](../06-chip-workload-analysis/transformer-workload.md)

## 这页在回答什么问题

这页回答 ViT 类 backbone 为什么改变了视觉模型的 workload 形态。CNN 把图像当成 feature map，ViT 把图像切成 token；这个表示变化会把硬件压力从规整卷积转向 patch embedding、GEMM、attention、token layout 和 activation memory。

## 从图像到 token

ViT 的基本做法是把图像切成固定大小 patch，每个 patch 展平成向量后投影成 token，再送入 Transformer encoder。

```text
image
   ->
patchify
   ->
linear projection / patch embedding
   ->
N visual tokens x C
   ->
Transformer encoder blocks
   ->
visual representation
```

如果输入是 `H x W`，patch size 是 `P x P`，token 数大约是 `(H/P) x (W/P)`。这意味着分辨率翻倍会让 token 数约增加 4 倍，而 full attention 的 score matrix 规模会约增加 16 倍。

例如 `224 x 224` 图像配 `16 x 16` patch 约产生 196 个 visual tokens；如果换成 `1024 x 1024` 且仍用 `16 x 16` patch，会产生 4096 个 tokens。这个量级差异解释了为什么高分辨率视觉系统必须使用分层结构、window attention 或 token 压缩。

ViT 的设计动机是用 Transformer 的全局建模能力替代 CNN 的固定局部归纳偏置。它不再假设邻域卷积是唯一合理的信息交互方式，而是让任意 patch token 可以通过 attention 建立关系。

## ViT 和 CNN 的本质差异

CNN 的 inductive bias 强：局部性、平移等变性、多尺度层次结构天然存在。ViT 的 inductive bias 弱：它更依赖数据规模、预训练和位置编码来学视觉结构。

从 workload 角度看，差异更直接：

| 维度 | CNN Backbone | ViT Backbone |
| --- | --- | --- |
| 基本表示 | `H x W x C` feature map | `N tokens x C` |
| 主算子 | Conv、BN/activation、elementwise | GEMM、attention、FFN、norm |
| 访问模式 | 规则滑窗，空间局部 | token matrix，attention 全局交互 |
| 复用 | weight/activation 空间复用强 | GEMM 复用强，attention 中间激活大 |
| 端侧风险 | depthwise 和分支碎片化 | token 数、activation memory、norm/layout |

常见误解：ViT 一定比 CNN 更“先进”。实际上，ViT 提供更统一的 token 表示和全局关系建模，但端侧小模型、小数据和强实时场景中，CNN 或 CNN+Transformer 混合结构仍可能更合适。

## 分层 ViT 与 Swin

原始 ViT 的 token 数在所有层基本不变，不天然产生 CNN 那样的多尺度 feature hierarchy。Swin Transformer 引入 window attention、shifted window 和 patch merging，让视觉 Transformer 更接近分层视觉 backbone。

这个设计解决了两个问题：第一，window attention 降低 full attention 的 `N^2` 成本；第二，patch merging 让后续层 token 数下降、channel 上升，形成多尺度表示。

代价是 workload 变得更复杂：window partition、shift、merge、reverse、relative position bias 都会引入 layout 和索引管理。对于芯片，Swin 的关键不是 attention FLOPs 低，而是这些 token 重排能否被高效调度。

## 在 AD、VLA、World Model 中的意义

自动驾驶中，ViT/Swin 可作为多摄像头 image encoder，也可提供更适合 query-based BEV 和 E2E 的 token 表示。VLA 中，ViT 是 vision encoder 的常见选择，因为它能把视觉输入转成和 language/action 更容易统一的 token。World Model 中，ViT 可以作为 frame encoder 或 latent tokenizer 的一部分。

但这些系统不一定需要 full-resolution ViT。端侧常见做法是降低 token 数、使用分层结构、用 CNN stem 或 lightweight vision encoder，避免视觉 token 直接压垮后端 VLM/VLA。

## 一句话理解

ViT backbone 把视觉 workload 从规整 feature map 卷积转成 token 序列交互；它提升了全局建模和多模态接口能力，但把压力转移到 token 数、attention activation、norm 和 layout 管理。

## Workload Characterization

- 计算密度：patch embedding 和 FFN 是高计算密度 GEMM；full attention 随 token 数平方增长，长序列下 activation memory 和带宽压力显著。
- 访存模式：GEMM 相对规整；attention score、softmax、token reshape、window partition 和 patch merge 会引入额外 activation 搬运。
- 并行性：token、head、hidden dimension、batch 可并行；端侧小 batch 下依赖 token/head 并行，长 token 序列会受 SRAM/DRAM 容量限制。
- 数据复用：projection/FFN weight 可复用；attention 可通过 tiling 复用 Q/K/V block；window attention 复用局部 token。
- 量化敏感度：linear/FFN 适合 INT8/FP8；LayerNorm、softmax、attention score、position bias 需要谨慎。
- 瓶颈类型：中小 token 数下偏 compute-bound；高分辨率、多帧、多相机场景容易 memory-capacity/bandwidth-bound；window/shift 操作可能 latency-bound。
- 对硬件的核心需求：高效 GEMM、attention tiling、norm fusion、layout transform、token/window 重排和 activation memory 管理。

## 参考来源

- Dosovitskiy et al., `An Image is Worth 16x16 Words: Transformers for Image Recognition at Scale`, ICLR 2021, arXiv:2010.11929。
- Liu et al., `Swin Transformer: Hierarchical Vision Transformer using Shifted Windows`, ICCV 2021, arXiv:2103.14030。

## 旧版素材

- `/mnt/e/workload-wiki-old/01_基础模型组件/Backbone/Backbone总览.md`
