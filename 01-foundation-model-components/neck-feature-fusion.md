# Neck Feature Fusion

上级：[Foundation Model Components](README.md)
相关：[CNN Backbone](cnn-backbone.md), [BEV Perception](../02-vision-and-3d-perception/bev-perception.md), [BEV Workload](../06-chip-workload-analysis/bev-workload.md)

## 这页在回答什么问题

这页回答 Neck 为什么是视觉系统里容易被低估的 workload。Backbone 负责提特征，Head 负责输出任务结果，Neck 位于二者之间，负责多尺度特征对齐、融合和重组。它的 FLOPs 未必最高，但经常引入大量 activation 搬运、上采样、concat 和分支同步。

## Neck 的计算结构

Backbone 输出的不同 stage 具有不同分辨率和语义层级。浅层 feature 分辨率高但语义弱，深层 feature 语义强但空间细节少。Neck 的作用是把这些特征整理成下游任务更容易消费的 multi-scale representation。

```text
backbone stage 2 / 3 / 4 / 5
   ->
lateral projection
   ->
top-down / bottom-up fusion
   ->
multi-scale feature pyramid
   ->
detection / segmentation / BEV / occupancy head
```

FPN 的典型结构是 top-down path + lateral connection。PANet/PAFPN 增加 bottom-up path，BiFPN 用可学习权重做双向融合。现代 BEV/E2E 系统中，Neck 的名字可能消失，但多尺度 feature alignment 和 fusion 仍然存在，只是被吸收到 scene encoder 或 BEV encoder 中。

## 为什么它对 workload 重要

Neck 不是单个大 GEMM，而是多分支数据重组。它常包含 1x1 conv、3x3 conv、upsample、downsample、elementwise add、concat、weighted sum 和 layout align。

这些操作的共同特点是：单个算子看起来轻，但它们频繁读写大 feature map，并且跨尺度同步。对于自动驾驶多摄像头前端，Neck 的输出还会进入 view transform 或 BEV encoder，因此 feature layout 会影响后续 DMA 和 cache 行为。

典型量级上，FPN 常处理 P3/P4/P5/P6 这类多尺度 feature，空间尺寸按 2 倍递减。浅层 feature 的 FLOPs 未必最高，但元素数量最大，因此 upsample、concat、add 的带宽压力常集中在高分辨率 level。

常见误解：Neck 是轻量模块，可以忽略。实际上，在端侧系统里，多尺度 feature map 的搬运、concat 和 resize 可能比部分卷积更影响 p99 latency，尤其在 multi-camera pipeline 中。

## 在自动驾驶和机器人中的位置

传统检测和分割直接依赖 FPN/PAFPN 来处理小目标、大目标和边界细节。自动驾驶更依赖多尺度，因为远距离目标和近场目标尺度差异很大。

在 BEV/Occupancy 中，Neck 决定送入 view transform 的 image feature 分辨率、channel 和尺度组合。高分辨率 feature 有利于几何细节，但会放大 camera-to-BEV 的采样和搬运成本；低分辨率 feature 成本低，但可能损失远处小目标和细粒度语义。

机器人 VLA 中，标准 FPN 未必显式出现，但多相机、多尺度对象特征仍需要被整理后送入 policy 或 VLM/VLA backbone。

## 演进路径

| 演进 | 设计动机 | Workload 变化 |
| --- | --- | --- |
| 单层特征 -> FPN | 小目标和多尺度语义不足 | 引入 top-down upsample、lateral 1x1 conv、add |
| FPN -> PANet / PAFPN | 只做 top-down 信息流不够 | 增加 bottom-up path，分支和中间 feature 增多 |
| PANet -> BiFPN | 融合路径重且权重固定 | 加权融合和重复堆叠，调度节点更多 |
| 显式 Neck -> Scene Encoder | BEV/E2E 需要统一多尺度、多视角、时序 | Neck 功能进入更大的 fusion block，访存和通信更复杂 |

## 一句话理解

Neck 是把 backbone 的多层 feature map 变成任务可用表示的重组层；它的核心 workload 不是大算力，而是多尺度 activation 搬运、对齐、融合和分支同步。

## Workload Characterization

- 计算密度：1x1/3x3 conv 有一定计算密度，但 upsample、add、concat、resize 多为 memory-bound；整体常不是纯 compute-bound。
- 访存模式：多尺度 feature map 读写频繁，concat 和 upsample 需要大 activation 搬运；若跨 camera 或跨尺度融合，layout 对 DMA 很敏感。
- 并行性：不同 feature level、camera、spatial tile 可并行；融合节点需要同步，top-down/bottom-up path 有顺序依赖。
- 数据复用：conv weight 可复用；activation 复用有限，主要依赖 on-chip buffer 保存相邻尺度和中间 feature。
- 量化敏感度：conv 和 elementwise 通常适合 INT8；resize/interpolation、weighted fusion 和后续 BEV 几何对齐需要关注精度。
- 瓶颈类型：multi-scale fusion 常 memory-bandwidth-bound 或 latency-bound；在多摄像头系统中可能成为 pipeline 同步瓶颈。
- 对硬件的核心需求：高效 2D DMA、feature map tiling、resize/concat/add 融合、multi-branch scheduling 和片上 buffer 管理。

## 参考来源

- Lin et al., `Feature Pyramid Networks for Object Detection`, CVPR 2017, arXiv:1612.03144。
- Liu et al., `Path Aggregation Network for Instance Segmentation`, CVPR 2018, arXiv:1803.01534。
- Tan et al., `EfficientDet: Scalable and Efficient Object Detection`, CVPR 2020, arXiv:1911.09070。
