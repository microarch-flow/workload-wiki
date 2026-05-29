# Contrastive Learning

上级：[Foundation Model Components](README.md)
相关：[JEPA and Self-supervised Representation](jepa-and-self-supervised.md), [CNN Backbone](cnn-backbone.md), [Vision Transformer Backbone](vision-transformer-backbone.md)

## 这页在回答什么问题

这页回答 Contrastive Learning 为什么是视觉表示预训练的重要起点，以及它对 workload 的意义是什么。这里不展开 loss 推导，而是关注“拉近正样本、拉远负样本”如何改变训练系统、表示质量和下游 backbone 的使用方式。

## 基本思想

Contrastive Learning 的核心是让同一语义对象的不同视图在 embedding space 中接近，让不同对象的表示分开。

```text
sample
   ->
augmentation view 1 / view 2
   ->
shared or momentum encoder
   ->
embeddings
   ->
contrastive objective
```

它解决的问题是：没有人工标签时，模型如何先学到可迁移表征。对于自动驾驶和机器人，这一点很重要，因为原始视频和传感器数据很多，但高质量标注昂贵。

常见误解：Contrastive Learning 是一个推理侧模型组件。实际上，它主要是训练范式；它对推理 workload 的影响是间接的，体现在 backbone 质量、token 表示、下游微调成本和模型规模选择上。

## 从 MoCo / SimCLR 到自蒸馏

MoCo 用 momentum encoder 和 queue 解决负样本数量问题，避免完全依赖极大 batch。SimCLR 强调强数据增强和大 batch，证明简单对比目标也能学到强表征。

BYOL、SimSiam 和 DINO 进一步弱化显式负样本依赖，转向 bootstrap/self-distillation。DINO 对 ViT 特别重要，因为它让自监督视觉 token 可以形成较强语义结构，影响后续 ViT backbone、VLM 和 VLA 的视觉 encoder。

这条路线的演进说明：表示预训练的重点从“如何构造负样本”逐步转向“如何让模型学到稳定、有语义结构的 embedding”。

从计算结构看，SimCLR 类方法会形成 batch 内 embedding similarity matrix。如果 batch size 是 `B`，两视图训练通常要比较 `2B x 2B` 的 embedding 相似度；多卡训练还需要 all-gather 其他设备上的 embeddings。MoCo 类方法用 queue/memory bank 保存历史 negative embeddings，降低对极大 batch 的依赖，但增加了队列读写和 momentum encoder 维护。

## 对自动驾驶和机器人有什么用

自动驾驶 camera 数据规模大，标注成本高。Contrastive/self-supervised pretraining 可以改善 image encoder 初始化，降低检测、分割、BEV、E2E 的标注依赖。

机器人也有大量无标注视频和操作观察。对比式或自蒸馏表征可以为 VLA 提供更稳的视觉 encoder，让语言/action head 不必从头学习底层视觉结构。

但是，这类方法本身不解决 closed-loop control，也不直接给出 world dynamics。它的定位是 representation foundation，而不是 policy 或 world model。

## Workload 影响

训练侧，Contrastive Learning 常常引入双视图、多分支 encoder、大 batch、negative queue、embedding similarity matrix 和跨设备同步。它比普通监督分类更依赖 batch 组织和 memory bank/queue。

推理侧，预训练完成后通常只保留 encoder，因此 runtime workload 仍由 CNN/ViT backbone 决定。架构师在端侧部署时不需要为 contrastive loss 建硬件，但需要知道这种训练范式会推动更强、更大的 encoder 进入下游系统。

## 一句话理解

Contrastive Learning 是学习可迁移视觉表示的训练路线；它主要改变上游训练 workload 和下游 backbone 质量，而不是直接增加端侧推理算子。

## Workload Characterization

- 计算密度：训练侧由双分支 encoder 和 embedding similarity 主导，通常 compute-heavy；推理侧回到普通 CNN/ViT encoder。
- 访存模式：训练需要保存多视图 activation、embedding、queue 或大 batch 表示；推理侧无额外 contrastive 访存。
- 并行性：训练可沿 batch/data parallel 扩展，但 embedding all-gather 和负样本组织会引入通信；推理侧按 backbone 并行。
- 数据复用：encoder weight 在两视图间共享；momentum encoder/queue 提供表示复用，但只存在训练侧。
- 量化敏感度：训练通常用 FP16/BF16；推理侧量化由 backbone 决定。
- 瓶颈类型：训练侧可能 compute + communication + memory capacity bound；端侧推理瓶颈不由 contrastive objective 决定。
- 对硬件的核心需求：云端训练需要高吞吐矩阵/卷积、跨设备通信和大 batch memory；端侧只继承 encoder 的 workload。

## 参考来源

- He et al., `Momentum Contrast for Unsupervised Visual Representation Learning`, CVPR 2020, arXiv:1911.05722。
- Chen et al., `A Simple Framework for Contrastive Learning of Visual Representations`, ICML 2020, arXiv:2002.05709。
- Caron et al., `Emerging Properties in Self-Supervised Vision Transformers`, ICCV 2021, arXiv:2104.14294。
