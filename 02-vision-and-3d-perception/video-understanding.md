# Video Understanding

上级：[Vision and 3D Perception](README.md)
相关：[Mamba and SSM](../01-foundation-model-components/mamba-and-ssm.md), [E2E Workload](../06-chip-workload-analysis/e2e-workload.md), [World Model Workload](../06-chip-workload-analysis/world-model-workload.md)

## 这页在回答什么问题

Video Understanding 回答模型如何从连续帧中理解运动、交互、历史上下文和未来趋势。它是自动驾驶 E2E、机器人 VLA、World Model 的时序感知底座。这页重点是时序建模的三种方式（3D conv / temporal attention / SSM）各自的 workload 性格，以及为什么"帧数 × 每帧 token 数"这个乘积决定了它能不能端侧跑。

## 为什么它有效：直觉与类比

单帧理解像**看一张照片**，视频理解像**看连环画推断剧情**——关键信息不在任何单张画里，而在画与画之间的变化：那辆车在动、那个杯子正被拿起、那个行人要横穿。所以视频模型的核心不是"再认一遍每帧有什么"，而是建模时间维上的关联。怎么建模这个时间关联，有三种直觉迥异的方式，它们的硬件性格也因此完全不同。

**3D conv** 的直觉是**拿一个时空小立方体滑过视频**——卷积核从 2D（高×宽）扩成 3D（高×宽×时间），一次只看相邻几帧的局部时空模式。它擅长抓短时局部运动（一个挥手动作），但只看局部、且成本随帧数和分辨率直接放大。**temporal attention**（TimeSformer/ViViT）的直觉是**让所有帧的所有 patch 互相开会**——每个 token 能 attention 到任意时刻的任意位置，建长依赖最强，但代价是 token 数 = 每帧 token × 帧数，落进 [Attention and Transformer](../01-foundation-model-components/attention-and-transformer.md) 那条 `N²` 爆炸链。**SSM/Mamba** 的直觉是**边看边在一张固定大小的便签上记笔记**（见 [Mamba and SSM](../01-foundation-model-components/mamba-and-ssm.md) 的便签类比）——流式处理每帧、更新状态、从不回看全部历史，特别适合长历史和在线 streaming，但要求硬件高效支持 scan/state update。

这三选一本质是在"建模能力、序列长度、硬件代价"之间权衡：要精确长程关联用 attention（贵）、要长历史流式用 SSM（需 scan 支持）、要短时局部用 3D conv（规整但视野窄）。

## 计算结构与决定性的乘积

```text
frames or video tokens -> frame encoder -> temporal model(3D conv / temporal attention / SSM / memory)
   -> video representation -> tracking / prediction / policy / world model
```

frame + temporal aggregation 易复用 image backbone 但空间时间分离；VideoMAE 类 masked video pretraining 说明大规模视频表征会成上游基础能力（和 [JEPA](../01-foundation-model-components/jepa-and-self-supervised.md) 一脉）。

决定端侧可行性的是这个乘积：**token 数 = 每帧 spatial tokens × history length**。把数字摆出来——每帧 1000 个 visual token、8 帧历史就是 8000 token，full temporal attention 的分数矩阵随这 8000 平方增长，迅速变成 activation memory 和带宽问题（同 [Vision Transformer Backbone](../01-foundation-model-components/vision-transformer-backbone.md) 的 token 爆炸，只是多乘了帧数这一维）。这解释了为什么长历史视频几乎不用 full temporal attention，而转向 window/factorized attention 或 SSM——把帧数这一维从平方里拆出来。

## 从视频理解到 World Model

传统视频理解关注动作识别、事件理解；自动驾驶和机器人更关心状态变化、目标运动、风险趋势、action-conditioned future。World Model 可看成视频理解的升级：不仅理解过去，还要在动作条件下预测未来（见 [World Model Fundamentals](../05-world-model-and-generative/world-model-fundamentals.md)）。常见误解：视频理解就是多帧图像分类。实际上在 AD/VLA/WM 中它是状态估计和未来预测的一部分，history cache、temporal alignment、latency 都是核心问题。

## 一句话理解

视频理解把静态视觉扩展成时序状态表示，靠 3D conv / temporal attention / SSM 三种方式建模帧间变化；它的 workload 由"每帧 token × 帧数"的乘积主导，long-horizon 下 full temporal attention 平方爆炸，逼着系统转向 window attention 或 SSM 的流式状态。

## 演进与未来方向（判断）

以下为判断，锚定真实趋势。

第一，**长历史时序建模正从 attention 倒向 SSM/hybrid 的流式状态**，这和 [Mamba and SSM](../01-foundation-model-components/mamba-and-ssm.md) 的判断互为表里。短视频片段 full attention 还扛得住，但自动驾驶/机器人要的是持续在线、历史可以无限长的 streaming 感知——每来一帧就把整个历史重算一遍 attention 既不现实也不必要。我的判断是在线时序感知会大量采用"固定大小状态 + 每帧增量更新"的范式（SSM 或带 memory 的 recurrent transformer），把 workload 从"随历史长度平方/线性增长"压成"每帧恒定"。对端侧这是关键转变——它让长历史感知的内存占用从随时间膨胀变成有界。

第二条对架构师最实际：**视频理解正在和 World Model 合流，从"理解过去"延伸到"预测未来"**，于是它的 workload 从一次性 forward 变成带状态的迭代推理。对 archax 的含义：在线视频/时序模型应建模为"frame encoder（compute-bound）+ 带状态的 temporal 更新（latency 敏感、stateful）"的复合工作点，其中 history cache 的容量与驻留策略、streaming state update 的延迟是关键——这和 [Mamba](../01-foundation-model-components/mamba-and-ssm.md) 的状态递推、[Diffusion](../01-foundation-model-components/diffusion-models.md) 的多步迭代同属"带状态的迭代推理"这一大类，是 06 的 [World Model Workload](../06-chip-workload-analysis/world-model-workload.md) 和 [E2E Workload](../06-chip-workload-analysis/e2e-workload.md) 共同要处理的时序维度。

## Workload Characterization

计算密度：frame encoder 可 compute-bound；temporal attention 随 token 数（帧 × 每帧 token）放大，长历史下转 memory-bound；SSM/temporal conv 的瓶颈取决于 scan kernel 和 state layout。

访存模式：多帧 activation 和 history cache 读写显著增加；temporal attention 需大 token matrix（`[帧×token]²`）；SSM 需顺序 state update；3D conv 是规整但体量大的时空访问。

并行性：frames、spatial tile、token 可并行；causal/streaming 模式沿时间有依赖，是并行断点。

数据复用：frame feature、temporal memory、BEV/video cache 可复用；长历史需要 cache eviction 策略（否则容量随时间无界增长）。

量化敏感度：frame encoder 可 INT8；temporal attention/SSM state 和 long-horizon prediction 需注意误差沿时间累积。

瓶颈类型：多帧前端常 bandwidth/capacity-bound；在线 policy 使用时常 latency-bound；full temporal attention 在长历史下被 `N²` 推向 memory-bound。

对硬件的核心需求：video feature cache、temporal token tiling、stateful execution（SSM/recurrent）、低延迟 streaming update、有界的 history cache 管理——详见 [World Model Workload](../06-chip-workload-analysis/world-model-workload.md)。

## 参考来源

- Tran et al., `Learning Spatiotemporal Features with 3D Convolutional Networks (C3D)`, ICCV 2015, arXiv:1412.0767。成熟度：已落地，3D conv 代表。
- Bertasius et al., `Is Space-Time Attention All You Need for Video Understanding? (TimeSformer)`, ICML 2021, arXiv:2102.05095。成熟度：已落地，时空 attention。
- Arnab et al., `ViViT: A Video Vision Transformer`, ICCV 2021, arXiv:2103.15691。成熟度：已落地，video transformer。
- Tong et al., `VideoMAE: Masked Autoencoders are Data-Efficient Learners for Self-Supervised Video Pre-Training`, NeurIPS 2022, arXiv:2203.12602。成熟度：已落地，视频自监督预训练。
- Gu and Dao, `Mamba: Linear-Time Sequence Modeling with Selective State Spaces`, arXiv:2312.00752。成熟度：已落地开源，长序列流式状态。
