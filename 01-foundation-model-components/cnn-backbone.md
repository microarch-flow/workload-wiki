# CNN Backbone

上级：[Foundation Model Components](README.md)
相关：[Vision Transformer Backbone](vision-transformer-backbone.md), [Neck Feature Fusion](neck-feature-fusion.md), [CNN Workload](../06-chip-workload-analysis/cnn-workload.md)

## 这页在回答什么问题

这页回答 CNN backbone 为什么长期是视觉感知和端侧推理的基础 workload。重点是理解卷积网络如何把图像变成多尺度 feature map，以及 standard conv、depthwise conv、pointwise conv、residual、FPN 接口分别意味着什么硬件行为。

## CNN Backbone 的计算结构

CNN backbone 从原始图像中提取层次化 feature map。早期 stage 保留较高空间分辨率，后期 stage 降低分辨率、增加 channel，并提取更强语义。

```text
image
   ->
conv stem
   ->
stage 1: high resolution / low semantics
   ->
stage 2
   ->
stage 3
   ->
stage 4: low resolution / high semantics
   ->
multi-scale features
```

标准卷积的核心是局部窗口和通道累加。对于输出位置 `(h, w, c_out)`，卷积读取输入局部窗口和对应 kernel 权重，做乘加得到输出。因为相邻输出位置共享大量输入像素，卷积天然有 activation reuse；因为同一组 kernel 在不同空间位置重复使用，卷积也有 weight reuse。

这就是 CNN 对 NPU 友好的根本原因：访问规则、复用稳定、shape 可预测、INT8 量化成熟。

## 从 ResNet 到轻量化 CNN

ResNet 的核心贡献是 residual connection。它解决深层 CNN 难训练的问题，让 backbone 可以堆得更深。对 workload 来说，ResNet 主要仍是规整 conv，但 residual add 引入跨层 activation 保存和逐元素加法。

MobileNet 等轻量化 CNN 使用 depthwise separable conv，把标准卷积分解成 depthwise conv 和 pointwise conv。它显著降低 FLOPs，但不一定等比例降低延迟，因为 depthwise conv 的计算量小、数据复用差，容易受 memory bandwidth 和 kernel overhead 限制；pointwise 1x1 conv 则更接近 GEMM，通常更适合 NPU。

典型量级上，3x3 standard conv 的每个 input activation 会被多个相邻输出复用，arithmetic intensity 通常明显高于 depthwise conv；depthwise conv 的 MAC 数约比 standard conv 少一个 `C_out` 量级，但 feature map 读写没有同等下降，所以低 FLOPs 不等于低延迟。

EfficientNet 使用 compound scaling 同时缩放 depth、width、resolution。它提醒架构师不要只看参数量：输入分辨率上升会放大 activation 和 bandwidth，channel 上升会提升 compute 和复用机会，depth 上升会增加 latency path。

常见误解：FLOPs 越低的 CNN 越适合端侧。实际上，如果 FLOPs 降低来自 depthwise、小 channel、分支碎片化和频繁 memory access，硬件利用率可能比一个稍重但规整的 1x1/3x3 conv 网络更差。

## CNN 在自动驾驶和机器人中的位置

在传统自动驾驶感知中，CNN backbone 是检测、分割、车道线和多任务 head 的前端。在 BEV/Occupancy 系统中，CNN 常作为多摄像头 image encoder，为 view transform 提供多尺度 feature。在 E2E、VLA、World Model 中，CNN 可以作为轻量 vision encoder，尤其适合端侧对 latency 和功耗敏感的场景。

CNN 的局限也很清楚：它擅长局部结构和多尺度特征，但全局关系、长时序、多模态融合不如 Transformer 自然。因此现代系统经常用 CNN 做前端特征提取，再用 Transformer、SSM 或 BEV encoder 做全局/时序建模。

## 设计动机到 workload 的演进

| 演进 | 设计动机 | Workload 变化 |
| --- | --- | --- |
| 浅层 CNN -> ResNet | 训练更深网络 | 更深规整 conv 栈，residual activation 保存更重要 |
| ResNet -> DenseNet | 强化特征复用 | concat 和跨层 activation 搬运增加 |
| Standard conv -> Depthwise separable conv | 降低 FLOPs 和参数量 | compute 下降，但 arithmetic intensity 下降，容易 memory-bound |
| 手工缩放 -> EfficientNet | 平衡 depth/width/resolution | activation、compute、latency 的放大来自不同参数 |
| CNN-only -> CNN + Transformer | 增强全局关系和多模态融合 | 前端规整 conv 与后端 token workload 混合 |

## 一句话理解

CNN backbone 是规整、高复用、量化成熟的视觉前端；它通常不是最强的全局关系建模结构，但仍然是端侧 NPU 最容易高效执行的基础 workload。

## Workload Characterization

- 计算密度：standard 3x3 conv 和 1x1 conv 在 channel 足够大时 FLOPs/Byte 较高，通常 compute-bound；depthwise conv FLOPs 低但复用弱，常 memory-bandwidth-bound。
- 访存模式：标准卷积访问规则、row locality 好，适合 tiling 和 DMA burst；residual、concat、multi-branch 会增加 activation 保存和额外读写。
- 并行性：可沿 batch、output channel、input channel、spatial tile 并行；端侧小 batch 下主要依赖 channel/spatial 并行。
- 数据复用：weight 在空间维复用，activation 在滑窗中复用；on-chip buffer 对 CNN 性能非常关键。
- 量化敏感度：CNN 是 INT8 最成熟的 workload 之一；first/last layer、depthwise、小 channel 和 detection/segmentation head 需要单独校准。
- 瓶颈类型：standard conv 多为 compute-bound；depthwise、FPN 多尺度融合和小 feature map stage 可能 memory/latency-bound。
- 对硬件的核心需求：高利用率 Conv/GEMM array、SRAM tiling、2D DMA、低开销 elementwise/concat、成熟 INT8 datapath。

## 参考来源

- He et al., `Deep Residual Learning for Image Recognition`, CVPR 2016, arXiv:1512.03385。
- Howard et al., `MobileNets: Efficient Convolutional Neural Networks for Mobile Vision Applications`, arXiv:1704.04861。
- Tan and Le, `EfficientNet: Rethinking Model Scaling for Convolutional Neural Networks`, ICML 2019, arXiv:1905.11946。
