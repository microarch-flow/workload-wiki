# Semantic Segmentation

上级：[Vision and 3D Perception](README.md)
相关：[Neck Feature Fusion](../01-foundation-model-components/neck-feature-fusion.md), [Occupancy Prediction](occupancy-prediction.md), [CNN Workload](../06-chip-workload-analysis/cnn-workload.md)

## 这页在回答什么问题

语义分割回答每个像素属于哪类区域。它把视觉理解从对象级 box 推到 dense scene understanding，是道路、可通行区域、地图语义、Occupancy、World Model 的基础监督之一。这页要讲清它的 workload 性格和检测为什么完全不同——检测输出稀疏对象，分割输出 dense field，硬件压力压在高分辨率输出的写回带宽上。

## 为什么它有效：直觉与类比

语义分割的直觉是**给整张图逐格涂色**，但难点是涂色既要"涂对类别"又要"涂准边界"，而这两件事天然矛盾。要知道一片像素是"道路"还是"人行道"，得看大范围上下文（语义需要大感受野，意味着要下采样、缩小看全局）；要把道路和路缘的边界涂准，又得保住高分辨率细节（边界需要小尺度，意味着不能下采样）。

encoder-decoder 结构就是为化解这个矛盾设计的，直觉是**先把图缩小看懂大意，再放大把标签精确涂回每个像素**。encoder 逐级下采样、扩感受野，得到"这一带是马路"的语义判断但丢了边界；decoder 逐级上采样恢复分辨率，并通过 skip connection 把 encoder 浅层留下的高分辨率细节接回来，让"马路"这个判断能精确贴回到马路和路缘的真实分界线上。对应到机制，skip connection 之所以关键，是因为深层语义和浅层边界是在不同分辨率上分别持有的，必须显式把二者重新对齐——这和 [Neck Feature Fusion](../01-foundation-model-components/neck-feature-fusion.md) 的多尺度融合是同一个道理，也带来同样的"轻计算、重搬运"特征。

DeepLab 的 atrous（空洞）卷积是另一个巧解，直觉是**把渔网的网眼撑大**：在不增加参数、不降分辨率的前提下，让卷积核覆盖更大范围——用稀疏采样换大感受野，避免了"下采样丢分辨率"和"堆深层加算力"两种代价。这解释了为什么它在需要高分辨率输出的分割里特别受用。

## 计算结构与那个写回带宽

```text
image -> encoder/backbone -> multi-scale features -> decoder(upsample + skip) -> pixel-wise logits
```

分割的 workload 签名是**高分辨率输出**。哪怕 backbone 不重，decoder、skip feature、upsampling、pixel-wise logits 都在高分辨率上反复读写，放大 activation memory。把数字摆出来：输出 `1024×512×20 classes` 的 logits，单帧就是约 1050 万个值，FP16 也有 20 MB 量级的写回压力，而且这是每帧、每个 decoder 尺度都要承受的搬运——所以分割常是 memory-bandwidth-bound，而非 compute-bound。自动驾驶的道路、路缘、车道、可通行区域要求边界稳定，输出分辨率不能压太低，于是实际系统靠低分辨率 logits + 轻量 decoder + 只在关键区域保高分辨率来换 latency。

常见误解：分割只是检测的像素版。实际上检测输出稀疏对象（几百个 box），分割输出 dense field（上千万个像素 logit），二者对 memory、decoder、输出带宽的要求完全不同——这是 02 章里两类最对照的 workload 性格。

## 演进与系统意义

语义分割从 FCN 到 encoder-decoder、context aggregation、Transformer/query segmentation，核心动机始终是同时拿到全局语义和局部边界。对 BEV，分割提供 image-plane dense semantics；对 Occupancy，它是从 2D dense label 走向 3D dense state 的前置概念（见 [Occupancy Prediction](occupancy-prediction.md)）。

## 一句话理解

语义分割把图像变成 dense semantic field，用"缩小看懂语义、放大涂准边界"的 encoder-decoder 化解语义与边界的矛盾；它的硬件压力不在算力，而在高分辨率 activation 的反复搬运和像素级 logits 的写回带宽。

## 演进与未来方向（判断）

以下为判断，锚定真实趋势。

主线是**逐像素分类（per-pixel softmax）正在被掩码分类（mask classification）统一取代**。Mask2Former 用"一组 query 各自预测一个 mask embedding，再和 pixel feature 做点积得到 mask"的方式，把语义分割、实例分割、全景分割统一进同一个 query-based 框架（见 [Instance Segmentation](instance-segmentation.md)）。我的判断是任务专用的分割 head 会继续让位给这种统一 mask transformer——和检测从 dense head 走向 query 是同一股潮流（见 [Object Detection](object-detection.md)）。对 workload 的含义：计算从"每像素 softmax over 类别"变成"query embedding × pixel feature 的 dense 乘法 + mask 上的归约"，但**高分辨率 mask 的写回带宽问题不变甚至更突出**——统一框架省了 head 设计，没省下那 20 MB 量级的 dense 输出搬运。

对架构师更实际的一条：在 AD 里语义分割正越来越多地不作为最终输出，而是作为 BEV/Occupancy 的中间监督和特征源。这意味着它的高分辨率输出可能不必真的写回 DRAM 给下游消费，而是 fuse 进后续的 BEV encoder——能否做这种 decoder-to-BEV 的算子融合、避免高分辨率 logits 落 DRAM，是端侧省带宽的关键，属于 archax 里 Interaction 轴上的融合决策，06 的 CNN/BEV workload 篇会接着分析。

## Workload Characterization

计算密度：encoder conv/attention 可能 compute-bound；decoder、upsample、pixel logits 常 memory-bandwidth-bound——分割的瓶颈天然偏向后者。

访存模式：feature map 访问规则但体量大；skip connection 和 multi-scale fusion 增加跨尺度读写；高分辨率 logits 写回是带宽大头。

并行性：spatial tile、channel、class logits 可并行；decoder stage 有分辨率逐级恢复的顺序依赖。

数据复用：encoder feature 可复用；decoder 的高分辨率 activation 体量大、片上驻留困难，易反复回 DRAM。

量化敏感度：conv/logits 可 INT8；边界、插值上采样、small class 的 logits 需校准（边界像素对量化误差敏感）。

瓶颈类型：memory bandwidth 和 activation capacity 常是第一瓶颈，尤其高分辨率输出；这与检测的"compute + 后处理同步"性格相反。

对硬件的核心需求：高效 upsample/concat/add、足够的 feature buffer、2D DMA、dense output 写回优化，以及 decoder 与 backbone（乃至下游 BEV）的融合调度以避免高分辨率中间结果落 DRAM——详见 [CNN Workload](../06-chip-workload-analysis/cnn-workload.md)。

## 参考来源

- Long et al., `Fully Convolutional Networks for Semantic Segmentation`, CVPR 2015, arXiv:1411.4038。成熟度：已落地，FCN 出处。
- Chen et al., `DeepLab: Semantic Image Segmentation with Atrous Convolution and Fully Connected CRFs`, TPAMI 2018, arXiv:1606.00915。成熟度：已落地，atrous 卷积代表。
- Cheng et al., `Masked-attention Mask Transformer for Universal Image Segmentation (Mask2Former)`, CVPR 2022, arXiv:2112.01527。成熟度：已落地，掩码分类统一框架。
