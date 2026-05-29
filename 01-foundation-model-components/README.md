# Foundation Model Components

上级：[Workload Wiki](../README.md)
相关：[Workload Lens](../00-overview/workload-lens.md), [Chip Workload Analysis](../06-chip-workload-analysis/README.md)

## 这章在回答什么问题

这一章回答“现代 AI workload 由哪些基础计算组件构成”。这里不把组件当成孤立算法名词，而是把它们看成后续视觉感知、自动驾驶 E2E、机器人 VLA 和 World Model 的 workload building blocks。

## 本章定位

01 是算法背景层，不显式展开 archax 建模；archax 和硬件 wiki 的连接放在 06。每篇文档都必须说明两件事：第一，组件的计算结构和设计动机；第二，它作为 workload 时会带来什么计算、访存、并行、复用和量化特征。

这章的核心关系是：

```text
CNN / ViT / Transformer / Diffusion / Mamba / SSL
        ->
视觉感知、BEV、E2E、VLA、World Model
        ->
06-chip-workload-analysis
```

## 阅读顺序

建议先读 [CNN Backbone](cnn-backbone.md) 和 [Attention and Transformer](attention-and-transformer.md)，建立“规整卷积”和“token 交互”两种最基本 workload。然后读 [Vision Transformer Backbone](vision-transformer-backbone.md) 和 [Attention Variants and Efficiency](attention-variants-and-efficiency.md)，理解为什么视觉系统从 CNN 走向 token，又为什么 full attention 必须被改造。最后读 [Diffusion Models](diffusion-models.md)、[Mamba and SSM](mamba-and-ssm.md)、[Contrastive Learning](contrastive-learning.md)、[JEPA and Self-supervised Representation](jepa-and-self-supervised.md)，把生成式、长序列和自监督表征纳入同一个 workload 视角。

## 页面列表

- [Attention and Transformer](attention-and-transformer.md)
- [Attention Variants and Efficiency](attention-variants-and-efficiency.md)
- [CNN Backbone](cnn-backbone.md)
- [Vision Transformer Backbone](vision-transformer-backbone.md)
- [Neck Feature Fusion](neck-feature-fusion.md)
- [Diffusion Models](diffusion-models.md)
- [Mamba and SSM](mamba-and-ssm.md)
- [Contrastive Learning](contrastive-learning.md)
- [JEPA and Self-supervised Representation](jepa-and-self-supervised.md)

## 本章到 06 的映射

| 组件 | 主要 workload 形态 | 06 对应入口 |
| --- | --- | --- |
| CNN Backbone | Conv、depthwise、pointwise、feature map reuse | [CNN Workload](../06-chip-workload-analysis/cnn-workload.md) |
| Transformer / ViT | GEMM、attention、FFN、norm、KV/cache | [Transformer Workload](../06-chip-workload-analysis/transformer-workload.md) |
| Attention variants | tiling、window、sparse、deformable sampling | [Transformer Workload](../06-chip-workload-analysis/transformer-workload.md), [BEV Workload](../06-chip-workload-analysis/bev-workload.md) |
| Neck / FPN | multi-scale feature fusion、upsample、concat | [CNN Workload](../06-chip-workload-analysis/cnn-workload.md), [BEV Workload](../06-chip-workload-analysis/bev-workload.md) |
| Diffusion | iterative denoising、multi-step sampling | [World Model Workload](../06-chip-workload-analysis/world-model-workload.md) |
| Mamba / SSM | selective scan、state update、streaming sequence | [Transformer Workload](../06-chip-workload-analysis/transformer-workload.md), [World Model Workload](../06-chip-workload-analysis/world-model-workload.md) |
| Contrastive / JEPA | 大规模预训练、embedding、representation prediction | 主要作为上游训练背景，推理侧影响体现在 backbone 和 world model |
