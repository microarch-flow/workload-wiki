# Video Understanding

上级：[Vision and 3D Perception](README.md)
相关：[Mamba and SSM](../01-foundation-model-components/mamba-and-ssm.md), [E2E Workload](../06-chip-workload-analysis/e2e-workload.md), [World Model Workload](../06-chip-workload-analysis/world-model-workload.md)

## 这页在回答什么问题

Video Understanding 回答模型如何从连续帧中理解运动、交互、历史上下文和未来趋势。它是自动驾驶 E2E、机器人 VLA 和 World Model 的时序感知底座。

## 计算结构

典型链路是：

```text
frames or video tokens
   ->
frame encoder
   ->
temporal model: 3D conv / temporal attention / SSM / memory
   ->
video representation
   ->
tracking / prediction / policy / world model
```

Frame + temporal aggregation 易于复用 image backbone，但空间和时间分离。3D conv 直接建模局部时空模式，但成本随帧数和分辨率放大。TimeSformer/ViViT 类 video transformer 用空间 attention、时间 attention 或 joint attention 建模视频 token；VideoMAE 类 masked video pretraining 说明大规模视频表征训练会成为上游基础能力。Temporal Transformer 建模长依赖强，但 token 数是 `spatial tokens x frames`。SSM/Mamba 更适合长历史和 streaming state update，但硬件需要高效 scan/state support。

典型量级上，如果每帧有 1000 个 visual tokens，8 帧历史就是 8000 tokens；full temporal attention 会迅速转成 activation memory 和 bandwidth 问题。

## 从视频理解到 World Model

传统视频理解关注动作识别和事件理解；自动驾驶和机器人更关心状态变化、目标运动、风险趋势和 action-conditioned future。World Model 可以看成视频理解的进一步升级：不仅理解过去，还要在动作条件下预测未来。

常见误解：视频理解就是多帧图像分类。实际上，在 AD/VLA/WM 中，视频理解是状态估计和未来预测的一部分，history cache、temporal alignment 和 latency 都是核心问题。

## 一句话理解

视频理解把静态视觉扩展成时序状态表示；它的 workload 由帧数、spatial tokens、history cache 和 temporal model 共同决定。

## Workload Characterization

- 计算密度：frame encoder 可 compute-bound；temporal attention 随 token 数放大；SSM/temporal conv 的瓶颈取决于 kernel 和 state layout。
- 访存模式：多帧 activation 和 history cache 读写显著增加；temporal attention 需要大 token matrix；SSM 需要 state update。
- 并行性：frames、spatial tiles、tokens 可并行；causal/streaming 模式有时间依赖。
- 数据复用：frame features、temporal memory、BEV/video cache 可复用；长历史需要 cache eviction 策略。
- 量化敏感度：frame encoder 可 INT8；temporal attention/SSM state 和 long-horizon prediction 需注意误差累积。
- 瓶颈类型：多帧前端常 bandwidth/capacity-bound；在线 policy 使用时常 latency-bound。
- 对硬件的核心需求：video feature cache、temporal token tiling、stateful execution、低延迟 streaming update。

## 参考来源

- Tran et al., `Learning Spatiotemporal Features with 3D Convolutional Networks`, ICCV 2015。
- Bertasius et al., `Is Space-Time Attention All You Need for Video Understanding?`, ICML 2021, arXiv:2102.05095。
- Arnab et al., `ViViT: A Video Vision Transformer`, ICCV 2021, arXiv:2103.15691。
- Tong et al., `VideoMAE: Masked Autoencoders are Data-Efficient Learners for Self-Supervised Video Pre-Training`, NeurIPS 2022, arXiv:2203.12602。
- Gu and Dao, `Mamba: Linear-Time Sequence Modeling with Selective State Spaces`, arXiv:2312.00752。
