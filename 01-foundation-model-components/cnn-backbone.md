# CNN Backbone

上级：[Foundation Model Components](README.md)
相关：[Vision Transformer Backbone](vision-transformer-backbone.md), [Neck Feature Fusion](neck-feature-fusion.md), [CNN Workload](../06-chip-workload-analysis/cnn-workload.md)

## 这页在回答什么问题

这页要把 CNN backbone 讲到能定量判断的程度：standard conv、depthwise conv、pointwise conv、residual 各自的 FLOPs 和算术强度是多少，为什么 MobileNet 的低 FLOPs 不等于低延迟，以及为什么 CNN 至今仍是端侧 NPU 最容易喂满的视觉前端。

## 为什么它有效：直觉与类比

卷积的直觉是**拿一个固定形状的小印章在整张图上滑动盖章**：同一个印章（kernel 权重）在每个位置重复使用，盖出来的是"这个局部长得像不像印章图案"的响应图。这个看似简单的设计藏着两个让它高效的归纳偏置。其一是**平移等变性**——一只猫在画面左上角还是右下角都是猫，印章在哪盖都用同一套权重，所以模型不必为每个位置单独学一遍，从很少的数据就能泛化（这正是 ViT 缺的先验，它得靠大数据补）。其二是**局部性**——自然图像的信息高度集中在邻域，先看小窗口是合理的起点。

层次结构的直觉是**先认笔画、再认偏旁、再认字、再读句**。浅层 stage 在高分辨率上提边缘和纹理，深层 stage 在低分辨率上提部件和物体语义——分辨率逐层减半、通道逐层加倍，把"空间细节"逐步换成"语义抽象"。对应到机制，这就是为什么 backbone 输出的是多尺度 feature：不同 stage 天然携带不同粒度的信息，下游小目标靠浅层、大目标靠深层。

residual 的直觉是**每层只学"该在上一层基础上改什么"，而不是从头重画一张**。学增量（残差）比学完整映射容易，且留了一条不被改写的直通路让梯度直达浅层——这是 ResNet 能从十几层堆到上百层的原因。

这些直觉合起来解释了 CNN 对硬件友好的根本：印章滑动 = 规则滑窗访问，印章复用 = 强 weight reuse，邻域重叠 = 强 activation reuse，shape 编译期可知 = 可预测 tiling。能力来自归纳偏置，效率来自同一套归纳偏置带来的规则性。

## 计算结构与真实的 FLOPs 账

一个 standard conv 层，输出 `H×W×C_out`、kernel `K×K`、输入通道 `C_in`，FLOPs 量级是 `H·W·C_out·C_in·K²`。关键是复用：每个输入像素被 `C_out` 个输出通道、`K²` 个空间位置反复读，所以算术强度高、是规则大计算，compute-bound，对 NPU 友好。ResNet-50 在 224×224 输入下约 25.6M 参数、4.1 GFLOPs，是端侧的常用锚点。

depthwise separable conv 把 standard conv 拆成两步：depthwise conv 每个通道各自做 `K×K` 空间卷积（`H·W·C_in·K²`），pointwise 1×1 conv 只混通道（`H·W·C_in·C_out`）。两者相加除以 standard 的 FLOPs，约等于 `1/C_out + 1/K²`——`K=3`、`C_out` 较大时约 `1/9`，所以 MobileNet 能把 FLOPs 砍到约 1/9。

但这里有个架构师必须看穿的陷阱：**FLOPs 砍了 9 倍，延迟往往砍不到**。depthwise conv 的每个输入像素只被 `K²=9` 次乘加复用（没有 `C_out` 那一维的复用），算术强度极低，feature map 的读写量却没等比例下降，于是它从 compute-bound 退化成 memory-bandwidth-bound——MAC 阵列大量空转等数据。pointwise 1×1 conv 反而更像规整 GEMM，通常更适合 NPU。所以 depthwise 网络在高算力低带宽的 NPU 上，实测利用率可能比一个稍重但规整的 3×3/1×1 网络还差。

常见误解：FLOPs 越低的 CNN 越适合端侧。实际上若低 FLOPs 来自 depthwise、小 channel、分支碎片化和频繁 memory access，硬件利用率可能更差；端侧选型要看的是"算术强度 × 访存规则性"，不是 FLOPs 单一数字。EfficientNet 的 compound scaling 同理提醒：分辨率上升放大 activation 和带宽，channel 上升提升算力和复用，depth 上升拉长 latency path——三者代价落在不同硬件维度，不能只看参数量。

## CNN 在自动驾驶与机器人中的位置

传统感知里 CNN backbone 是检测、分割、车道线、多任务 head 的前端。BEV/Occupancy 系统里 CNN 常作多摄像头 image encoder，为 view transform 提供多尺度 feature（见 [Neck Feature Fusion](neck-feature-fusion.md)、[BEV Perception](../02-vision-and-3d-perception/bev-perception.md)）。E2E/VLA/World Model 里 CNN 可作轻量 vision encoder，尤其在 latency 和功耗敏感的端侧。它的局限也清楚：擅长局部结构与多尺度，但全局关系、长时序、多模态融合不如 Transformer 自然，所以现代系统常是 CNN 做前端、Transformer/SSM/BEV encoder 做全局与时序。

## 一句话理解

CNN backbone 是规则、高复用、INT8 成熟的视觉前端；standard/1×1 conv 是 compute-bound 的规整计算，depthwise 是 memory-bound 的低强度计算——选型的关键不是 FLOPs，而是算术强度和访存规则性能不能喂满目标 NPU。

## 演进与未来方向（判断）

以下为判断，锚定真实趋势。

第一，**CNN 没有被 ViT 取代，而是分工固化**。ConvNeXt（arXiv:2201.03545）证明用现代训练配方，纯卷积能追平同规模 ViT，说明 CNN 的退场是被误判的。我的判断是端侧低延迟视觉前端在可见的未来仍由 CNN 或 CNN-hybrid 主导——理由是 INT8 通路成熟、访存规则、shape 静态可调度，这些在功耗和确定性受限的车端/机器人上是硬优势；纯 ViT 的动态大 activation 在这类场景反而吃亏。

第二，**效率前沿正持续往 depthwise/grouped 这类低强度算子走**，而这恰好是 NPU 最不擅长的区域。这给硬件设计提了一个真问题：一颗为高强度 dense conv 优化（高 MAC:带宽比）的 NPU，跑 depthwise 时会严重带宽饿死。我的判断是未来端侧视觉 NPU 必须支持可变 dataflow，能在 dense conv 的 weight-stationary 和 depthwise 的 activation-stationary 之间切换，而不是锁死一种 MAC:带宽配比。对 archax，这意味着 CNN 不能当成单一 workload 点评估，至少要把 standard-conv-heavy 和 depthwise-heavy 两类作为 Resource/Interaction 轴上分开的工作点——这和 06 对 CNN workload 的拆法一致。

## Workload Characterization

计算密度：standard 3×3 conv 和 1×1 conv 在 channel 足够大时算术强度高（每输入像素被 `C_out·K²` 次复用），compute-bound；depthwise conv 每像素仅 `K²` 次复用，强度低，常 memory-bandwidth-bound——这是 CNN 内部最大的两类性格分裂。

访存模式：标准卷积规则滑窗、row locality 好，适合 tiling 和 2D DMA burst；residual、concat、multi-branch 增加 activation 保存和额外读写；depthwise 的逐通道访问对 layout（NCHW vs NHWC）敏感。

并行性：可沿 batch、output channel、input channel、spatial tile 并行；端侧小 batch 下主要靠 channel/spatial 并行，depthwise 因无 `C_out` 复用维并行效率天然低。

数据复用：weight 在空间维复用、activation 在滑窗中复用，是 CNN 高效的根；on-chip buffer 容量直接决定能否把复用兑现，depthwise 的复用机会本就少，buffer 帮助有限。

量化敏感度：CNN 是 INT8 最成熟的 workload 之一；first/last layer、depthwise、小 channel、detection/segmentation head 需单独校准。

瓶颈类型：standard/1×1 conv 多 compute-bound；depthwise、FPN 多尺度融合、小 feature map stage 多 memory/latency-bound。

对硬件的核心需求：高利用率 Conv/GEMM 阵列、SRAM tiling、2D DMA、低开销 elementwise/concat、成熟 INT8 datapath，以及能兼顾高强度与低强度算子的可变 dataflow——详见 [CNN Workload](../06-chip-workload-analysis/cnn-workload.md)。

## 参考来源

- He et al., `Deep Residual Learning for Image Recognition`, CVPR 2016, arXiv:1512.03385。成熟度：已落地，residual 出处。
- Howard et al., `MobileNets: Efficient Convolutional Neural Networks for Mobile Vision Applications`, arXiv:1704.04861。成熟度：已落地，depthwise separable 代表。
- Tan and Le, `EfficientNet: Rethinking Model Scaling for Convolutional Neural Networks`, ICML 2019, arXiv:1905.11946。成熟度：已落地，compound scaling。
- Liu et al., `A ConvNet for the 2020s (ConvNeXt)`, CVPR 2022, arXiv:2201.03545。成熟度：已落地，现代卷积追平 ViT 的代表。
