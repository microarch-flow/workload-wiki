# CNN Workload

上级：[Chip Workload Analysis](README.md)
相关：[CNN Backbone](../01-foundation-model-components/cnn-backbone.md), [Workload Characterization Axes](workload-characterization-axes.md)

## 这页在回答什么问题

这页把 CNN 翻译成可建模的芯片 workload。CNN 的关键不在"卷积计算量大"，而在不同卷积形态的**数据复用结构截然不同**：standard conv 是 deterministic NPU 最理想的对象，而 depthwise/group conv 会把同一颗芯片从 compute-bound 推向 memory-bound。理解这个分化，才能解释为什么 MobileNet 这类"低 FLOPs"网络在 NPU 上反而难以达到峰值利用率。

## standard conv 为什么是 NPU 最友好的 workload

一个标准卷积层的乘加量约为 `H × W × Cout × Cin × Kh × Kw`，而它要读的数据是权重 `Cout × Cin × Kh × Kw` 加上输入 activation `H × W × Cin`。这里有三重复用同时发生：权重在输出空间 `H × W` 上被反复使用（每个输出位置都用同一组 kernel），activation 在 `Cout` 个输出通道之间被共享，相邻输出位置的滑窗又让输入 activation 在 kernel window 内重叠复用。

正是这三重复用让 standard conv 的 arithmetic intensity 很高：一份权重 tile 加载进 SRAM 后能服务整张 feature map，一份 activation tile 能喂给所有输出通道。因此只要 tiling 设计得当（weight stationary 或 output stationary），standard conv 能稳定喂满 MAC 阵列，是典型的 **compute-bound**，也是 INT8 量化收益最直接的场景。

## depthwise / group conv 为什么相反

depthwise conv 把每个输入通道单独卷积，不跨通道求和。乘加量降到约 `H × W × C × Kh × Kw`——比 standard conv 少了 `Cout`（或 `Cin`）这个因子，看起来"省了一个数量级算力"。但代价是：每个通道只有自己那一组小 kernel，权重的输出通道复用没有了；activation 仍要按通道全部读写，读写量并没有等比例下降。

结果是 arithmetic intensity 暴跌：MAC 数下降一个数量级，但访存量几乎不变，于是 FLOPs/Byte 掉到很低，depthwise conv 从 compute-bound 变成 **memory-bandwidth-bound**。这解释了一个反直觉现象——MobileNet/EfficientNet 这类用 depthwise 把理论 FLOPs 压得很低的网络，在很多 NPU 上的实测利用率反而比 ResNet 低，因为 MAC 阵列被 activation 搬运卡住，喂不满。1×1 conv（pointwise）则又回到规则 GEMM，复用充分、compute-bound，所以 MobileNet 的实际算力大头其实落在 1×1 上。

## 早层和后处理：另两类容易被忽略的成本

stem 和早期层处理的 feature map 最大（例如 `224×224` 输入、stride 后仍是 `112×112`、`56×56` 量级），channel 还少，所以这些层是 **activation 带宽和 SRAM tile 容量**主导，而非权重算力——它们决定了片上 buffer 要开多大、early-layer latency 有多高。

检测/分割的 head 和 NMS、ROI、mask 上采样这类后处理则是小 tensor、低 batch、强 latency 的算子；如果放在 CPU 上做，会在 NPU 流水里插入同步点，成为 p99 延迟的隐藏来源。这类成本在只看 backbone FLOPs 时完全看不到。

## 可建模参数

`H × W` 决定 activation 容量、DMA 搬运量和早层延迟；`Cin / Cout` 决定 MAC 数、权重复用强度和 channel 并行度；`kernel size` 决定 activation 复用、halo buffer 大小和 FLOPs；`stride / dilation` 决定访存的规则性与 row locality；`group count`（从 standard 到 depthwise 是 group 从 1 到 C 的连续谱）决定 MAC 利用率和权重复用是否成立；`batch` 决定权重复用，端侧通常 batch=1，权重复用只能靠空间维度而非 batch 维度获得。

## 硬件连接

standard conv 的规则 strided 访问对 RAM 的 row locality 友好，早层大 feature map 需要 bank parallelism 和 SRAM tiling（见 RAM wiki）。DMA 需要 2D stride、halo 区搬运和 double buffering；FPN/neck 还需要多尺度 feature 的 producer-consumer 调度（见 DMA wiki）。当 output channel 或 spatial tile 跨多个 NPU tile 切分时，NOC 要提供清晰的 reduction/concat 路径（见 NOC wiki）。standard conv、1×1 conv、im2col 化的卷积都是 CIM wiki 最适合的对象；而 depthwise、resize、postprocess 在 CIM 上收益有限，因为它们要么复用不足、要么不是规则矩阵运算。端侧 CNN 推理应避免频繁回 host，后处理若在 CPU 上会形成同步点（见 PCIE wiki）。

## archax 建模

Resource：MAC array TOPS、SRAM 容量、DRAM bandwidth、DMA channel，可选 CIM array——其中 depthwise-heavy 网络的关键资源是 DRAM bandwidth 而非 TOPS。Topology：权重/activation 在 tile 内复用，多尺度 feature 在 tile 间流动。Interaction：`input tile → conv → activation tile → next block` 的流水，FPN 是多 producer 汇到 fusion stage。Capability：Conv2D、1×1 GEMM、depthwise/group conv、INT8/FP8 累加、2D DMA stride。架构探索时应把 group count 作为一个显式扫描维度，因为它直接决定该层落在 compute-bound 还是 bandwidth-bound。

## 一句话理解

CNN 的 workload 画像不取决于总 FLOPs，而取决于卷积形态的复用结构：standard/1×1 conv 复用充分、compute-bound、NPU 友好；depthwise/group conv 复用坍塌、memory-bound，"低 FLOPs"反而难喂满阵列。

## Workload Characterization

- 计算密度：standard conv、1×1 conv 高，compute-bound；depthwise/group conv 低，memory-bandwidth-bound；早层受 activation 带宽主导。
- 访存模式：standard conv 规则 strided、row locality 好；FPN 的 resize/concat 增加额外 activation 搬运；depthwise 读写量不随 MAC 下降。
- 并行性：batch、channel、spatial tile 都可并行；端侧 batch=1 时只能靠 channel/spatial 并行获得吞吐。
- 数据复用：standard conv 的权重（输出空间）/activation（滑窗+通道）三重复用强；depthwise 复用弱；早层 activation 容量压力大。
- 量化敏感度：INT8 在主干上成熟；first/last layer、small-channel depthwise、检测/分割 head 需单独验证。
- 瓶颈类型：标准 backbone 多为 compute-bound；MobileNet/depthwise/FPN 多为 memory-bandwidth 或 latency-bound。
- 对硬件的核心需求：高利用率 Conv/GEMM array、SRAM tiling、2D DMA、INT8/FP8 datapath、低开销 fusion；对 depthwise-heavy 网络则首先是 DRAM 带宽。

## 参考来源

- He et al., `Deep Residual Learning for Image Recognition`, CVPR 2016 / arXiv:1512.03385。成熟度：已落地基础 backbone。
- Howard et al., `MobileNets: Efficient Convolutional Neural Networks for Mobile Vision Applications`, arXiv:1704.04861。成熟度：已落地，depthwise 范式代表。
- Tan and Le, `EfficientNet: Rethinking Model Scaling for Convolutional Neural Networks`, ICML 2019 / arXiv:1905.11946。成熟度：已落地。
- Chen et al., `Eyeriss: An Energy-Efficient Reconfigurable Accelerator for Deep Convolutional Neural Networks`, ISSCC/JSSC 2016。成熟度：已落地，卷积数据复用与 dataflow 分析的经典参考。
