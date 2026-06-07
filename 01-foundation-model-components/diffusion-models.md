# Diffusion Models

上级：[Foundation Model Components](README.md)
相关：[Diffusion for World Model](../05-world-model-and-generative/diffusion-for-world-model.md), [World Model Workload](../06-chip-workload-analysis/world-model-workload.md)

## 这页在回答什么问题

这页回答 Diffusion 为什么从图像生成扩展到视频、轨迹、动作和 World Model，重点是看懂"多步去噪"这种计算结构——它把 workload 从一次 forward 变成 iterative inference，于是真正决定能否端侧部署的不是单步网络的 FLOPs，而是 `单步 × 步数 × 候选数 × 时空/动作 horizon` 这个乘积。

## 为什么它有效：直觉与类比

Diffusion 的直觉是**把"凭空画一张精细的画"这个难到没法一步做对的问题，拆成"在一团噪点上反复擦一点、改一点"的几十次小修正**。正向过程是往清晰图像里一点点加噪，直到变成纯噪声；模型学的是这个过程的逆——给一张带噪的图，预测"该去掉哪些噪声"。推理时从纯噪声起步，反复问模型"现在该擦掉什么"，每问一次图像就清晰一档，几十步后浮现出目标。

为什么这比"一步生成"有效：直接让网络从噪声一步映射到清晰图像，等于要它一口气拟合极其复杂的高维分布，容易崩。拆成多步后，每一步只需做一个**局部、平滑、好学**的小去噪——把一个陡峭的难题改造成一串缓坡的易题。这也解释了 Diffusion 为什么特别擅长多峰分布：当一个场景有多种合理的未来、一个指令有多种合理的动作时，逐步去噪能自然地走向其中一个模式而不是把它们平均成糊状（这正是回归模型的通病），所以它在动作/轨迹生成上有独特价值。

代价就藏在"反复"两个字里。同一个去噪网络要被调用几十次，每次都是一次完整 forward——能力来自多步，负担也来自多步。这是 Diffusion 一切 workload 分析的起点：单步再便宜，乘上步数和候选数也可能端侧跑不动。

## 计算结构与那个决定性的乘积

```text
noise / noisy latent
   -> denoiser step 1 -> step 2 -> ... -> step T
   -> generated sample / trajectory / action / future latent
```

核心 workload 参数不是单步 denoiser 的 FLOPs，而是：

```text
total cost = denoiser cost × sampling steps × candidates （× 时空/动作 horizon）
```

经典 DDPM 采样要数百到上千步；实际部署用 DDIM、DPM-Solver、distillation、consistency model 把步数压到几十步甚至个位数。对端侧 policy 或 World Model，`step count` 经常比单步网络 FLOPs 更先决定可部署性——一个单步只有几 GFLOPs 的小 denoiser，乘上 50 步和 8 个候选就是几百 GFLOPs 的端侧实时预算问题。

常见误解：Diffusion 的 workload 等于 U-Net/Transformer denoiser 的单步 workload。实际上必须按 step count、candidate count、horizon、条件输入一起算，否则会数量级地低估端侧延迟和云端仿真成本。

## 三个部署变体：把成本压在不同地方

Latent Diffusion 不在像素空间去噪，而在压缩 latent 空间去噪，单步张量规模大降（Stable Diffusion 在 64×64 latent 上去噪而非 512×512 像素），代价是引入 VAE encoder/decoder，且 latent 压缩率与语义保真度会影响下游 World Model/视频质量。Video Diffusion 把时间维纳入去噪，张量变成 `T×H×W×C` 或 latent video token，更接近 World Model 但 workload 重得多，还要维持时间一致性。Policy/Action Diffusion（如 Diffusion Policy）用去噪生成轨迹或动作序列，适合多峰动作分布，但机器人/自动驾驶端侧必须严控采样步数，因为 action policy 有控制频率和闭环延迟的硬约束——这点详见 [VLA Fundamentals](../04-robotics-and-vla/vla-fundamentals.md) 与 [VLA Workload](../06-chip-workload-analysis/vla-workload.md)。

## 与 Autoregressive 生成的硬件性格对比

Autoregressive 逐 token 生成，瓶颈是 KV cache 和 per-token latency，硬件性格是 stateful decode（见 [Transformer Workload](../06-chip-workload-analysis/transformer-workload.md) 的 decode 阶段）。Diffusion 逐 denoising step 生成，每步可并行处理整段 latent/action，但步间有顺序依赖，硬件性格是 repeated full forward。二者瓶颈不同：AR 是窄而深的带宽问题，Diffusion 是宽而重复的计算问题。World Model 里二者常混合——例如 Transformer 做 denoiser 或 autoregressive rollout，再用 diffusion 生成未来帧。

## 一句话理解

Diffusion 用多步去噪把"一步难题"换成"多步易题"，质量和多峰建模能力由此而来，端侧成本也由此而来——它的 workload 必须按 `单步 × 步数 × 候选 × horizon` 的乘积分析，步数往往比单步 FLOPs 更决定可部署性。

## 演进与未来方向（判断）

以下为判断，锚定真实趋势。

主线是**整个领域在拼命砍步数，把 Diffusion 从"多步"逼向"少步乃至一步"**。DDIM 把随机采样改确定性、DPM-Solver 用高阶常微分求解器、consistency model（arXiv:2303.01469）直接学"任意噪声一步到清晰"、各类 distillation 把几十步教师蒸成几步学生。我的判断是端侧 Diffusion 的未来是"4 步以内"甚至单步——一旦步数降到个位数，那个决定性乘积里最大的放大因子被拆掉，Diffusion 的端侧 workload 就从"重复 forward"塌缩回"接近一次 forward"，硬件需求随之从 latency-bound 回到单次网络的 compute-bound。对架构师这意味着：评估 Diffusion 端侧可行性时，`step count` 是头号可变参数，且它在未来 2-3 年会持续下探，不能用今天的 50 步去判死刑。

第二条判断，**denoiser 主干正从 U-Net 转向 Transformer（DiT，arXiv:2212.09748）**，这让 Diffusion 的单步 workload 和本章其他 Transformer 篇收敛——同样是 GEMM + attention + norm，同样吃 attention 的 `N²` 和 activation memory。好处是硬件不必为 Diffusion 单独优化一套算子，坏处是视频/高分辨率 DiT 的 token 数会把 attention 成本顶上去。对 archax，Diffusion 应建模为"一个可参数化 denoiser（DiT/U-Net）× 可变步数 × 候选数"的复合工作点，其中步数是 Interaction 轴上必须显式扫描的维度——这正是 06 的 [World Model Workload](../06-chip-workload-analysis/world-model-workload.md) 展开的。

## Workload Characterization

计算密度：单步 denoiser（U-Net 或 DiT）本身可 compute-bound；总成本被 step count 线性放大，低步数方案显著改变端侧可行性——这是 Diffusion 区别于普通 forward 网络的核心。

访存模式：U-Net 类有多尺度 activation 和 skip connection（类似 Neck 的多尺度搬运）；DiT 类有 attention/FFN activation 和 `N²` 中间张量；多步采样反复读写 latent state。

并行性：单步内部可并行，多个 candidate 可并行；denoising step 之间顺序依赖，不能跨步并行——这是 Diffusion 并行性的硬上限。

数据复用：condition embedding、text/vision context、static scene feature 可跨 step 复用（一次算好缓存）；latent state 每步更新，需高效驻留或搬运。

量化敏感度：denoiser 的 Conv/GEMM 可低比特，但多步误差会累积放大，低比特需验证长采样链的稳定性；scheduler、variance、最终 action/trajectory 输出需谨慎。

瓶颈类型：云端高分辨率生成可能 compute-bound；端侧 policy diffusion 常 latency-bound（步数 × 控制频率）；video/world-model diffusion 还可能 memory-capacity-bound（`T×H×W` latent）。

对硬件的核心需求：高效 repeated inference、latent state buffer、condition cache（跨步复用条件）、candidate 并行、低延迟 step scheduling、低比特 denoiser 支持——详见 [World Model Workload](../06-chip-workload-analysis/world-model-workload.md)。

## 参考来源

- Ho et al., `Denoising Diffusion Probabilistic Models`, NeurIPS 2020, arXiv:2006.11239。成熟度：已落地，DDPM 出处。
- Song et al., `Denoising Diffusion Implicit Models`, ICLR 2021, arXiv:2010.02502。成熟度：已落地，DDIM 确定性少步采样。
- Rombach et al., `High-Resolution Image Synthesis with Latent Diffusion Models`, CVPR 2022, arXiv:2112.10752。成熟度：已落地，latent diffusion。
- Song et al., `Consistency Models`, ICML 2023, arXiv:2303.01469。成熟度：研究到落地之间，少步/一步生成代表。
- Peebles and Xie, `Scalable Diffusion Models with Transformers (DiT)`, ICCV 2023, arXiv:2212.09748。成熟度：已落地，Transformer denoiser 主干。
