# Neck Feature Fusion

上级：[Foundation Model Components](README.md)
相关：[CNN Backbone](cnn-backbone.md), [BEV Perception](../02-vision-and-3d-perception/bev-perception.md), [BEV Workload](../06-chip-workload-analysis/bev-workload.md)

## 这页在回答什么问题

这页回答一个容易被低估的事实：Neck 的 FLOPs 通常不高，却经常是端侧 pipeline 的延迟元凶。它负责把 backbone 的多尺度 feature 对齐、融合、重组，主要算子是 upsample、concat、add、1×1 conv——单个看都轻，但反复读写大 feature map、跨尺度同步，带宽和 p99 latency 压力集中在这里。这页要讲清它为什么重要、代价压在哪。

## 为什么它有效：直觉与类比

Neck 的直觉是**一个翻译协调员，坐在专家组（backbone）和决策者（head）之间**。backbone 的浅层 stage 像近视的细节专家——看得清边缘纹理但说不出这是什么；深层 stage 像远视的语义专家——知道"这是车"但说不准边界在哪。决策者要同时要细节和语义，但两个专家说的是不同分辨率的"语言"。Neck 干的就是把深层的语义"翻译"回高分辨率（top-down upsample），把浅层的细节"翻译"上去（bottom-up），让每个尺度都既有细节又有语义。

为什么这有效：小目标只在高分辨率浅层留得下痕迹，大目标只在低分辨率深层有完整语义，单靠任一层都会漏一头。FPN 的 top-down + lateral 把深层语义注入浅层，让高分辨率 feature 也带上语义，于是同一个检测头能同时管好远处的小车和近处的大车——对应到机制，这就是为什么自动驾驶离不开多尺度，因为远近目标的尺度差异是数量级的。

但翻译协调员的工作量不在"想"而在"搬"。把语义图放大到高分辨率、再和细节图拼接相加，搬的是整张大 feature map——这是个体力活不是脑力活，FLOPs 不高但访存极重。这个"轻计算、重搬运"的本质，是 Neck 一切 workload 特征的根。

## 计算结构与代价分布

backbone 不同 stage 分辨率和语义层级不同，Neck 把它们整理成下游易消费的多尺度表示：

```text
backbone stage 2 / 3 / 4 / 5
   -> lateral 1×1 projection（统一 channel）
   -> top-down upsample + add（语义注入高分辨率）
   -> （PANet 额外 bottom-up path）
   -> multi-scale feature pyramid -> head
```

代价分布的关键在于：FLOPs 和元素数量落在不同的 level。深层 feature 通道多但空间小，FLOPs 可能不低但元素少；浅层 feature 通道少但空间大，FLOPs 不高但**元素数量最大**，于是 upsample、concat、add 的带宽压力集中在高分辨率 level。举例，P3 相对 P5 空间大 16 倍，同样一次 add 在 P3 上搬的字节数就是 P5 的 16 倍——这就是为什么"看起来最轻"的浅层融合反而最吃带宽。

FPN 是 top-down + lateral；PANet/PAFPN 加 bottom-up path，分支和中间 feature 增多；BiFPN 用可学习权重做双向融合并重复堆叠，调度节点更多。现代 BEV/E2E 系统里 Neck 这个名字可能消失，但多尺度对齐与融合被吸收进 scene encoder / BEV encoder，访存特征不变甚至更重。

常见误解：Neck 轻量、可忽略。实际上端侧多摄像头 pipeline 里，多尺度 feature 的搬运、concat、resize 可能比部分卷积更影响 p99 latency，且常成为分支同步的瓶颈点。

## 在自动驾驶与机器人中的位置

传统检测/分割直接靠 FPN/PAFPN 处理小目标、大目标和边界。BEV/Occupancy 里 Neck 决定送入 view transform 的 image feature 分辨率、channel、尺度组合：高分辨率利于几何细节但放大 camera-to-BEV 的采样与搬运成本，低分辨率便宜但丢远处小目标——这个取舍直接影响 [BEV Workload](../06-chip-workload-analysis/bev-workload.md) 的访存量。机器人 VLA 里标准 FPN 未必显式出现，但多相机、多尺度对象特征仍需整理后送入 policy 或 VLA backbone。

## 一句话理解

Neck 是把 backbone 多层 feature 重组成任务可用表示的"翻译协调员"，它的核心 workload 不是算力而是多尺度大 feature map 的搬运、对齐、融合与分支同步——轻计算、重访存，瓶颈压在高分辨率 level。

## 演进与未来方向（判断）

以下为判断，锚定真实趋势。

主线是**显式 Neck 正在溶解进统一的 scene/BEV encoder**。从 FPN 到 PANet 到 BiFPN 是"融合路径越加越密"，再往后 BEV/E2E 系统干脆把多尺度融合、多视角融合、时序融合合并进一个大 attention/transformer fusion block（如 BEVFormer 的 spatial/temporal cross-attention）。我的判断是独立 Neck 模块会越来越少，但它代表的那类 workload——低算术强度、大 activation 搬运、多分支同步——不会消失，只会从"FPN 的 concat/add"变成"fusion block 里的 cross-attention KV 搬运和 deformable 采样"，访存压力甚至更大。

对架构师这条判断的价值在于：不要因为 Neck 在新架构里"看不见名字"就在 workload 建模里漏掉它。无论叫 FPN 还是叫 BEV encoder，pipeline 里始终有一段是 memory-bound 的多尺度搬运，它和 compute-bound 的 backbone 必须作为**两个不同性格的工作点**分别评估——一颗只为高算术强度卷积优化的 NPU，会在这段融合上带宽饿死。对 archax，这段融合在 Interaction 轴上是"搬运主导"的极端点，是 backbone（计算主导）的对照面，二者不能用平均值合并。这与 [CNN Backbone](cnn-backbone.md)、[Vision Transformer Backbone](vision-transformer-backbone.md) 提到的"同一 pipeline 内性格分裂"是同一个道理。

## Workload Characterization

计算密度：1×1/3×3 conv 有一定算术强度，但 upsample、add、concat、resize 多为低强度 memory-bound 算子；整体很少是纯 compute-bound，常被搬运拖住。

访存模式：多尺度 feature map 读写频繁，concat/upsample 搬运大 activation；高分辨率 level 元素数最大、带宽压力最集中；跨 camera/跨尺度融合时 layout 对 DMA 极敏感。

并行性：不同 feature level、camera、spatial tile 可并行；融合节点需同步，top-down/bottom-up path 有顺序依赖，是 pipeline 同步点。

数据复用：conv weight 可复用；activation 复用有限，主要靠 on-chip buffer 暂存相邻尺度和中间 feature，buffer 不够就反复回 DRAM。

量化敏感度：conv 和 elementwise 通常适合 INT8；resize/interpolation、weighted fusion、后续 BEV 几何对齐需关注精度。

瓶颈类型：multi-scale fusion 常 memory-bandwidth-bound 或 latency-bound；多摄像头系统中易成 pipeline 同步瓶颈，影响 p99 而非平均延迟。

对硬件的核心需求：高效 2D DMA、feature map tiling、resize/concat/add 的算子融合、multi-branch scheduling、足够的片上 buffer 暂存多尺度中间结果——这些在 06 落到 RAM 带宽与 DMA 调度，是和 backbone 算力需求正交的另一类需求。

## 参考来源

- Lin et al., `Feature Pyramid Networks for Object Detection`, CVPR 2017, arXiv:1612.03144。成熟度：已落地，FPN 出处。
- Liu et al., `Path Aggregation Network for Instance Segmentation`, CVPR 2018, arXiv:1803.01534。成熟度：已落地，PANet/PAFPN。
- Tan et al., `EfficientDet: Scalable and Efficient Object Detection (BiFPN)`, CVPR 2020, arXiv:1911.09070。成熟度：已落地，加权双向融合。
