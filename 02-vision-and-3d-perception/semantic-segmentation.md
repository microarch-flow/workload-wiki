# Semantic Segmentation

上级：[Vision and 3D Perception](README.md)
相关：[Neck Feature Fusion](../01-foundation-model-components/neck-feature-fusion.md), [Occupancy Prediction](occupancy-prediction.md), [CNN Workload](../06-chip-workload-analysis/cnn-workload.md)

## 这页在回答什么问题

语义分割回答每个像素属于哪类区域。它把视觉理解从对象级 box 推向 dense scene understanding，是道路、可通行区域、地图语义、Occupancy 和 World Model 的基础监督之一。

## 计算结构

典型链路是：

```text
image -> encoder/backbone -> multi-scale features -> decoder -> pixel-wise logits
```

Encoder 压缩空间分辨率并增强语义，decoder 通过 upsample、skip connection、multi-scale fusion 恢复空间细节。DeepLab 这类路线用 atrous convolution 和 context aggregation 扩大感受野，Transformer segmentation 则用 token 交互增强全局语义。

分割的 workload 特征是高分辨率输出。即使 backbone 不重，decoder、skip feature、upsampling 和 pixel-wise logits 都会放大 activation memory。自动驾驶中道路、路缘、车道、可通行区域要求边界稳定，因此不能简单把输出分辨率降得太低。

典型量级上，如果输出是 `1024 x 512 x 20 classes` logits，单帧 logits 就超过 1000 万个值；即使用 FP16 也有二十 MB 级别的写回压力。实际系统常用低分辨率 logits、轻量 decoder 或只在关键区域保留高分辨率，以换取 latency。

## 演进与系统意义

语义分割从 FCN 到 encoder-decoder、context aggregation、Transformer/query segmentation，核心动机是同时获得全局语义和局部边界。对 BEV 来说，分割提供 image-plane dense semantics；对 Occupancy 来说，它是从 2D dense label 走向 3D dense state 的前置概念。

常见误解：分割只是检测的像素版。实际上，检测输出稀疏对象，分割输出 dense field；两者对 memory、decoder 和输出 bandwidth 的要求完全不同。

## 一句话理解

语义分割把图像变成 dense semantic field；它的硬件压力集中在高分辨率 activation、decoder upsampling 和像素级输出。

## Workload Characterization

- 计算密度：encoder conv/attention 可能 compute-bound；decoder、upsample、pixel logits 常 memory-bandwidth-bound。
- 访存模式：feature map 访问规则但体量大；skip connection 和 multi-scale fusion 增加读写。
- 并行性：spatial tile、channel、class logits 可并行；decoder stage 有分辨率逐级恢复的依赖。
- 数据复用：encoder feature 可复用；decoder 中高分辨率 activation 驻留困难。
- 量化敏感度：conv/logits 可 INT8；边界、插值、small class logits 需要校准。
- 瓶颈类型：memory bandwidth 和 activation capacity 常是第一瓶颈，尤其高分辨率输出。
- 对硬件的核心需求：高效 upsample/concat/add、feature buffer、2D DMA、dense output 写回优化，以及 decoder 与 backbone 的融合调度。

## 参考来源

- Long et al., `Fully Convolutional Networks for Semantic Segmentation`, CVPR 2015。
- Chen et al., `DeepLab: Semantic Image Segmentation with Deep Convolutional Nets, Atrous Convolution, and Fully Connected CRFs`, TPAMI 2018, arXiv:1606.00915。
- Cheng et al., `Masked-attention Mask Transformer for Universal Image Segmentation`, CVPR 2022, arXiv:2112.01527。

## 旧版素材

- `/mnt/e/workload-wiki-old/02_视觉与3D感知/语义分割/语义分割总览.md`
