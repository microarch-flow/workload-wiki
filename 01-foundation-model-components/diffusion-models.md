# Diffusion Models

上级：[Foundation Model Components](README.md)
相关：[Diffusion for World Model](../05-world-model-and-generative/diffusion-for-world-model.md), [World Model Workload](../06-chip-workload-analysis/world-model-workload.md)

## 这页在回答什么问题

这页回答 Diffusion 为什么从图像生成扩展到视频、轨迹、动作和 World Model。重点不是训练目标推导，而是理解“多步去噪”这种计算结构，以及它为什么会把 workload 从单次 forward 变成 iterative inference。

## Diffusion 的计算结构

Diffusion 把复杂生成问题拆成一系列去噪步骤。训练时，模型学习在不同噪声强度下预测噪声、残差或干净样本；推理时，从噪声开始，重复调用 denoiser，逐步得到目标样本。

```text
noise / noisy latent
   ->
denoiser step 1
   ->
denoiser step 2
   ->
...
   ->
generated sample / trajectory / action / future latent
```

这个结构的核心 workload 参数不是单次 denoiser 的 FLOPs，而是：

```text
total cost = denoiser cost x sampling steps x candidates
```

如果生成对象是视频、BEV future、occupancy future 或 robot action chunk，还要乘上时间维、空间维或 action horizon。

典型量级上，经典 DDPM 采样可以有数百到上千步；实际部署会用 DDIM、solver、distillation 或 consistency 类方法把步数压到几十步甚至更少。对端侧 policy 或 World Model，`step count` 经常比单步网络 FLOPs 更先决定是否可部署。

## 为什么它质量高但推理重

Diffusion 的优势来自把复杂分布建模拆成很多局部更简单的问题。多峰未来、复杂动作分布、视觉细节和场景变化都可以通过逐步去噪表达。

代价也来自同一个设计：推理需要重复调用网络。DDIM、DPM-Solver、distillation、latent diffusion 等方法都在减少采样步数或降低单步张量规模，但只要仍然是多步采样，就必须面对累计延迟。

常见误解：Diffusion 的 workload 等于 U-Net 或 Transformer denoiser 的单步 workload。实际上，部署时必须按 step count、candidate count、horizon 和条件输入一起计算，否则会低估端侧延迟和云端仿真成本。

## Latent Diffusion、Video Diffusion 和 Policy Diffusion

Latent Diffusion 不在像素空间去噪，而是在压缩 latent 空间去噪。它降低单步张量规模，但引入 encoder/decoder，并且 latent 的压缩率和语义保真度会影响后续 World Model 或视频生成质量。

Video Diffusion 把时间维纳入 denoising。它比图像 diffusion 更接近 World Model，但 workload 更重，因为张量通常是 `T x H x W x C` 或 latent video tokens，并且需要保持时间一致性。

Policy / Action Diffusion 用 diffusion 生成轨迹或动作序列。它适合多峰动作分布，但机器人或自动驾驶端侧部署必须控制采样步数，因为 action policy 有控制频率和闭环延迟约束。

## 与 Autoregressive 生成的区别

Autoregressive 逐 token 生成，瓶颈经常是 KV cache 和 per-token latency。Diffusion 逐 denoising step 生成，每一步可以并行处理整段 latent/action，但需要多次迭代。

因此二者的硬件压力不同：AR 更像 stateful decode，Diffusion 更像 repeated full forward。对于 World Model，二者可能混合出现，例如 video-action model 用 Transformer 做 denoiser 或 autoregressive rollout，再用 diffusion 生成未来。

## 一句话理解

Diffusion 把生成质量换成多步去噪计算；它的 workload 必须按单步 denoiser、采样步数、候选数和时空/动作 horizon 一起分析。

## Workload Characterization

- 计算密度：单步 denoiser 如果是 U-Net/Transformer，可能 compute-bound；总成本由 step count 放大，低步数方案会显著改变端侧可行性。
- 访存模式：U-Net 类结构有多尺度 activation 和 skip connection；Transformer denoiser 有 attention/FFN activation；多步采样会反复读写 latent state。
- 并行性：单步内部可并行，多个 candidate 可并行；denoising steps 通常有顺序依赖，不能完全并行。
- 数据复用：condition embedding、text/vision context、static scene feature 可跨 step 复用；latent state 每步更新，需要高效驻留或搬运。
- 量化敏感度：denoiser 的 Conv/GEMM 可低比特，但多步误差会累积；scheduler、variance、final action/trajectory 输出需要谨慎。
- 瓶颈类型：云端生成可能 compute-bound；端侧 policy diffusion 常 latency-bound；video/world model diffusion 还可能 memory-capacity-bound。
- 对硬件的核心需求：高效 repeated inference、state buffer、condition cache、candidate parallel、低延迟 step scheduling 和低比特 denoiser 支持。

## 参考来源

- Ho et al., `Denoising Diffusion Probabilistic Models`, NeurIPS 2020, arXiv:2006.11239。
- Song et al., `Denoising Diffusion Implicit Models`, ICLR 2021, arXiv:2010.02502。
- Rombach et al., `High-Resolution Image Synthesis with Latent Diffusion Models`, CVPR 2022, arXiv:2112.10752。
