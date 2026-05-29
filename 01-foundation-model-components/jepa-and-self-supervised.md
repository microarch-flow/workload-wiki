# JEPA and Self-supervised Representation

上级：[Foundation Model Components](README.md)
相关：[Contrastive Learning](contrastive-learning.md), [World Model Fundamentals](../05-world-model-and-generative/world-model-fundamentals.md), [World Model Workload](../06-chip-workload-analysis/world-model-workload.md)

## 这页在回答什么问题

这页回答 JEPA 为什么是从视觉自监督走向 World Model 的关键中间层。它的重点不是重建像素，而是预测高层表征；因此它比普通 masked reconstruction 更接近“学习世界中可预测的结构”。

## JEPA 的计算结构

JEPA，Joint-Embedding Predictive Architecture，核心思想是 predict representation, not pixels。模型看到 context region，预测 target region 的高层 embedding；target encoder 给出真实 target embedding，训练时让预测表示与目标表示接近。

```text
context input
   ->
context encoder
   ->
context representation
   ->
predictor
   ->
predicted target representation

target input
   ->
target encoder
   ->
target representation
```

它和 MAE 的关键区别是预测对象不同。MAE 倾向于重建被遮挡的原始信号或像素，JEPA 预测抽象表示。因为像素细节可能迫使模型关注纹理和局部噪声，而自动驾驶、机器人和 World Model 更关心物体、空间关系、时序趋势和可行动结构。

常见误解：JEPA 就是另一种 masked autoencoder。实际上，JEPA 的目标不是恢复输入本身，而是让模型在 embedding space 中学习上下文到目标的语义预测关系。

## I-JEPA、V-JEPA 和 V-JEPA 2

I-JEPA 把 JEPA 用于图像，证明不靠像素重建也能学习高质量视觉表示。V-JEPA 把这种思想扩展到视频，通过时空上下文预测缺失或未来片段的表示。

V-JEPA 2 在 2025 年进一步把 self-supervised video representation 和 physical world understanding、prediction、planning 联系起来，并展示了 action-conditioned latent world model 的机器人规划用法。V-JEPA 2.1 在 2026 年继续强调 dense video/image features，更适合作为 perception、tracking、segmentation 等密集视觉任务的通用表征来源。它们的成熟度仍应理解为研究系统和开源模型阶段，不应写成量产机器人方案。

典型量级上，视频 JEPA 的输入不再是单张图像，而是多个 frame 或 tubelet token。history window、frame resolution、patch/tubelet size 会共同决定 token 数；因此它的训练成本更接近 video foundation model，而不是普通 image self-supervised pretraining。

这条线对本 wiki 很重要，因为 World Model 需要预测未来，但并不一定要预测像素级未来。对于芯片 workload，latent representation prediction 可能比 raw video generation 更适合端侧风险评估和短 horizon planning。

## 对 World Model 的意义

World Model 可以有多种预测对象：pixel video、latent state、BEV、occupancy、action-conditioned future。JEPA 强调在表示空间预测目标，这为 latent world model 提供了思想基础。

如果预测对象是高维像素，workload 容易被 decoder 和视频生成细节主导；如果预测对象是 latent representation，计算可能更集中在 encoder、predictor 和 latent dynamics 上。后者更可能服务自动驾驶/机器人中的 planning、risk evaluation 和 state rollout。

## Workload 影响

训练侧，JEPA 需要 context encoder、target encoder、predictor，以及 mask/target 采样策略。视频 JEPA 还会引入长时序 token 和大规模无标注视频 batch。

推理侧，JEPA 本身常作为预训练表示或 world model latent predictor 的基础。若用于在线 World Model，它的 workload 会表现为 latent state update、future representation prediction 和 action-conditioned rollout，而不是像素 decoder。

## 一句话理解

JEPA 把自监督学习从“对比或重建输入”推进到“预测高层表征”，它是理解 latent world model 和 decision-oriented prediction 的重要桥梁。

## Workload Characterization

- 计算密度：encoder/predictor 多为 CNN/ViT/Transformer，训练侧 compute-heavy；latent prediction 比 pixel reconstruction 的 decoder 压力更低。
- 访存模式：训练需要保存 context/target 表示和视频 token；视频版本的时空 token 会放大 activation memory。
- 并行性：image JEPA 可沿 batch/patch 并行；video JEPA 可沿 batch、spatial token、部分 temporal chunk 并行，但长时序预测有依赖。
- 数据复用：context representation、target embedding 和 latent state 可复用；world model 场景中可跨 rollout 复用压缩状态。
- 量化敏感度：推理侧 encoder/predictor 可尝试低比特；训练侧和表征预测目标通常需要 FP16/BF16 保持稳定。
- 瓶颈类型：训练侧多为 compute + activation memory；在线 latent world model 可能转为 latency + state cache。
- 对硬件的核心需求：高效 video/ViT encoder、latent state buffer、predictor GEMM、temporal token 管理和 action-conditioned state update。

## 参考来源

- Assran et al., `Self-Supervised Learning from Images with a Joint-Embedding Predictive Architecture`, CVPR 2023, arXiv:2301.08243。
- Bardes et al., `Revisiting Feature Prediction for Learning Visual Representations from Video`, 2024, arXiv:2404.08471。
- `V-JEPA 2: Self-Supervised Video Models Enable Understanding, Prediction and Planning`, 2025, arXiv:2506.09985。
- `V-JEPA 2.1: Dense Video and Image Features`, 2026, arXiv:2603.14482。
