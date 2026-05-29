# Diffusion for World Model

上级：[World Model and Generative Intelligence](README.md)
相关：[Diffusion Models](../01-foundation-model-components/diffusion-models.md), [World Model Workload](../06-chip-workload-analysis/world-model-workload.md)

## 这页在回答什么问题

这页回答 Diffusion 为什么常用于 World Model，以及它对 workload 的特殊影响。Diffusion 擅长建模多模态未来，因为同一个当前状态可能对应多个合理未来；但它的迭代 denoising 会显著增加推理成本。

## 基本形式

```text
condition: history + action + map + task
   ->
noise latent
   ->
iterative denoising network
   ->
future video / trajectory / occupancy / action chunk
```

Diffusion 的价值在于分布建模。自动驾驶中，其他车辆可能让行、加速或变道；机器人中，同一个抓取任务可能有多条可行动作轨迹。Diffusion 可以生成多个 candidate，再由 cost/risk/policy 选择。

## 为什么不是免费收益

Diffusion 的问题是迭代。即使用 latent diffusion，每个 sample 也要经过多个 denoise step；如果还要生成多个 candidate future，计算成本会乘上 sample 数。对云端仿真这可以接受，对端侧实时 planning 则必须压缩步数、降低分辨率或改用 distillation/flow matching。

Flow matching 和 consistency distillation 是 2024-2026 很重要的方向：它们试图减少采样步数，让生成式 world model 更接近实时。

## 在不同输出上的差异

| 输出 | diffusion 价值 | workload 代价 |
| --- | --- | --- |
| video | 高保真、多样未来 | 最大的时空 latent 和 decoder 成本 |
| occupancy | 多假设占用风险 | 3D grid 容量大，边界敏感 |
| trajectory/action | 多模态动作候选 | sample 数和实时性冲突 |
| latent state | 高效多未来预测 | 可解释性较弱 |

## 一句话理解

Diffusion 让 World Model 能表达多种未来，但也把 workload 从一次前向变成多步生成和多样本评估。

## Workload Characterization

- 计算密度：denoising backbone 反复执行，计算量约随 step 数和 sample 数线性增长。
- 访存模式：每步需要读写 latent、condition、KV/activation；高分辨率视频或 3D occupancy 会显著放大容量。
- 并行性：sample/candidate 可并行；denoise step 有串行依赖；classifier-free guidance 会增加条件分支计算。
- 数据复用：condition encoding 可在多个 denoise step 和 sample 间复用，是优化关键。
- 量化敏感度：denoise backbone 可混合精度；最后几步、边界细节、collision/contact 相关输出需要更谨慎。
- 瓶颈类型：训练侧是显存与吞吐；推理侧是 step latency、sample 数和 condition cache。
- 对硬件的核心需求：高效重复执行、condition cache 复用、多样本并发、低步数生成支持、视频/3D latent tiling。

## 参考来源

- Ho et al., `Denoising Diffusion Probabilistic Models`, NeurIPS 2020，https://arxiv.org/abs/2006.11239，成熟度：基础方法，查证日期：2026-05-29。
- Rombach et al., `High-Resolution Image Synthesis with Latent Diffusion Models`, CVPR 2022，https://arxiv.org/abs/2112.10752，成熟度：latent diffusion 基础，查证日期：2026-05-29。
- Chi et al., `Diffusion Policy`, RSS 2023 / arXiv:2303.04137，https://arxiv.org/abs/2303.04137，成熟度：机器人动作生成成熟研究，查证日期：2026-05-29。
- NVIDIA, `Cosmos World Foundation Models`, 2025，https://www.nvidia.com/en-us/ai/cosmos/，成熟度：2025 physical AI world model 平台，查证日期：2026-05-29。

## 旧版素材

- `/mnt/e/workload-wiki-old/05_World_Model与生成式智能/Diffusion_for_World_Model.md`
