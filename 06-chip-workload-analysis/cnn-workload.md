# CNN Workload

上级：[Chip Workload Analysis](README.md)
相关：[CNN Backbone](../01-foundation-model-components/cnn-backbone.md), [Workload Characterization Axes](workload-characterization-axes.md)

## 这页在回答什么问题

这页把 CNN 从算法结构翻译成芯片 workload。CNN 的关键不是“卷积计算量大”，而是卷积窗口、channel、feature map 和数据复用方式共同决定 MAC 利用率、SRAM 需求、DMA 搬运和量化收益。

## Stage 拆解

| Stage | 典型算子 | 主导成本 |
| --- | --- | --- |
| stem / early conv | large feature map、3x3/7x7 conv | activation 带宽与 SRAM tile |
| backbone block | standard conv、1x1 conv、depthwise conv | standard conv compute-bound，depthwise memory-bound |
| neck / FPN | resize、concat、1x1/3x3 conv | 多尺度 feature 搬运 |
| head | classifier、box/mask/logit head | 小 tensor、低 batch latency |

standard conv 的复用最适合 deterministic NPU：weight 在 output channel 上复用，activation 在 kernel window 和 output spatial 上复用。depthwise conv 则相反，MAC 数显著下降，但每个 channel 独立，activation 读写没有等比例下降，容易从 compute-bound 变成 memory-bandwidth-bound。

## 关键参数

| 参数 | 放大什么 |
| --- | --- |
| `H x W` | activation 容量、DMA 搬运、early layer latency |
| `Cin / Cout` | MAC 数、weight 复用、channel 并行度 |
| `kernel size` | activation reuse、halo buffer、FLOPs |
| `stride / dilation` | 访存规则性、row locality |
| `group count` | MAC 利用率和 weight reuse |
| `batch` | weight reuse，端侧通常 batch=1 |

## 硬件连接

- RAM：standard conv 对 row locality 友好；early feature map 大，需要 bank parallelism 和 SRAM tiling。
- DMA：需要 2D stride、halo 搬运、double buffering；FPN 还需要多尺度 feature 的 producer-consumer 调度。
- NOC：output channel 或 spatial tile 跨 NPU tile 分发时，需要清晰的 reduction/concat 路径。
- CIM：standard conv、1x1 conv、large GEMM 化卷积适合 CIM；depthwise、resize、postprocess 收益有限。
- PCIE/host：端侧 CNN 推理通常不应频繁回 host；后处理如果在 CPU 上做，会形成同步点。

## archax 建模

- Resource：MAC array TOPS、SRAM 容量、DRAM bandwidth、DMA channel、可选 CIM array。
- Topology：weight/activation 在 tile 内复用，multi-scale feature 在 tile 间流动。
- Interaction：`input tile -> conv -> activation tile -> next block` 的流水；FPN 是多 producer 到 fusion stage。
- Capability：Conv2D、1x1 GEMM、depthwise/group conv、INT8/FP8 accumulation、2D DMA stride。

## Workload Characterization

- 计算密度：standard conv 和 1x1 conv 高，通常 compute-bound；depthwise/group conv 计算密度低，常 memory-bound。
- 访存模式：standard conv 规则 strided，row locality 好；FPN resize/concat 增加额外 activation 搬运。
- 并行性：batch、channel、spatial tile 都可并行；端侧 batch=1 时主要依赖 channel/spatial 并行。
- 数据复用：standard conv 的 weight/activation 复用强；depthwise 复用弱；early layer activation 容量压力大。
- 量化敏感度：INT8 CNN 成熟；first/last layer、small channel depthwise、检测/分割 head 需要验证。
- 瓶颈类型：标准 backbone 多为 compute-bound；MobileNet/Depthwise/FPN 多为 memory-bandwidth 或 latency-bound。
- 硬件核心需求：高利用率 Conv/GEMM array、SRAM tiling、2D DMA、INT8/FP8 datapath、低开销 fusion。

## 参考来源

- He et al., `Deep Residual Learning for Image Recognition`, CVPR 2016。
- Howard et al., `MobileNets: Efficient Convolutional Neural Networks for Mobile Vision Applications`, arXiv:1704.04861。
- Tan and Le, `EfficientNet`, ICML 2019。
- `/mnt/e/workload-wiki-old/06_芯片架构Workload分析/CNN_Workload.md`。
