# Diffusion for World Model

上级：[World Model and Generative Intelligence](README.md)
相关：[Diffusion Models](../01-foundation-model-components/diffusion-models.md), [Video World Model](video-world-model.md), [Latent World Model](latent-world-model.md), [World Model Workload](../06-chip-workload-analysis/world-model-workload.md)

## 这页在回答什么问题

这页回答 Diffusion 当 World Model 的 denoiser 为什么有独特价值，以及它带来的那个核心矛盾：多步采样 vs 实时 rollout。Diffusion 擅长建模多模态未来（同一现在对应多个合理未来），但它的迭代去噪把 World Model 本已是乘法的成本再乘上采样步数，于是 `candidate × horizon × steps` 三重相乘——理解这个三重乘法，才知道为什么 diffusion world model 偏云端，以及砍步数为什么是它能否上端侧的头号变量。

## 为什么它有效：直觉与类比

Diffusion 当 World Model 的价值，来自它能**让未来"开枝散叶"而不是被平均成一团糊**。前方路口其他车可能让行、加速或变道，机器人同一抓取任务有多条可行轨迹——未来本质是多峰的。回归式模型预测单一未来，会把这些模式平均成一个谁都不像的中间态（一条穿过所有可能性的灰色轨迹）。Diffusion 从噪声逐步去噪，每次采样自然走向其中一个模式，能生成多个 candidate future 交给 cost/risk/policy 挑选。对应到机制，diffusion 学的是条件分布 `p(future | history, action)` 的得分，多步去噪是在这个分布上采样而非取均值，所以天然保住多峰性。这正是 [Diffusion Models](../01-foundation-model-components/diffusion-models.md) 里"逐步去噪走向一个模式而非平均"的能力，落到 World Model 上。

为什么这个能力对 World Model 特别重要：World Model 的用途是"预演多个候选动作的后果选最安全"，而每个动作下的未来本身又是多峰的——diffusion 让模型能为同一 action 采样出"前车让行"和"前车加塞"两种未来分别评估风险，而不是赌一个均值。对应到机制，candidate action 与 per-action 的多 sample 是两层并行，diffusion 在第二层提供多样性。

代价就藏在"逐步"里：同一个 denoiser 要被调用几十次，每次都是完整 forward。能力来自多步，负担也来自多步——这是 diffusion world model 一切 workload 分析的起点。

## 三重乘法：World Model 的乘法又被乘上步数

```text
condition: history + action + map + task
   -> noise latent
   -> denoiser × N 步去噪  [步间顺序依赖]
   -> future video / occupancy / trajectory / action chunk
```

普通推理是一次 forward；World Model 已是 `candidate × horizon × per-step`（见 [World Model Fundamentals](world-model-fundamentals.md)）；diffusion 把 per-step 本身变成"几十步去噪的一次完整生成"，于是总成本是：

```text
total ≈ denoiser cost × sampling steps × candidates × horizon
```

经典 DDPM 采样数百到上千步；DDIM、DPM-Solver、consistency、distillation、flow matching 把步数压到几十乃至个位数。量级直觉：一个单步几 GFLOPs 的 denoiser，乘 50 步、8 candidate、几步 horizon，就是几千 GFLOPs 级的单次成本。这个三重乘法直接决定了——云端仿真/数据生成承受得起（不受实时约束，可大 batch 摊薄），端侧实时 planning 必须把 steps 压到个位数、降分辨率，或干脆改用非迭代表示。这与 06 [World Model Workload](../06-chip-workload-analysis/world-model-workload.md) 的乘法爆炸是同一件事的极端情形。

## 在不同输出上的差异：同一矛盾，不同代价

| 输出 | diffusion 的价值 | workload 代价 |
| --- | --- | --- |
| video | 高保真、多样未来 | 最大的 T×H×W latent + 像素 decoder + 多步采样，三重乘法顶满（见 [Video World Model](video-world-model.md)） |
| occupancy | 多假设占用风险 | 3D grid 容量大、边界敏感，采样步数 × voxel 容量 |
| trajectory / action | 多模态动作候选 | sample 数与控制频率/闭环延迟硬冲突（Diffusion Policy） |
| latent state | 高效多未来预测 | 最轻，但可解释性弱（见 [Latent World Model](latent-world-model.md)） |

同一个"多步采样 vs 实时"的矛盾，在 video 上是 capacity 顶满，在 action 上是和控制频率（机器人/驾驶有几十到上百 Hz 的硬约束）正面冲突，在 latent 上最轻。这解释了为什么 Diffusion Policy（arXiv:2303.04137）这类端侧动作生成必须严控采样步数——闭环延迟容不下几十步去噪。

## 一句话理解

Diffusion 当 World Model 的 denoiser，用多步去噪换来多峰未来建模（不把可能性平均成糊），但也把 World Model 的 `candidate × horizon` 乘法再乘上采样步数，形成三重乘法；这让它偏云端，而采样步数是它能否上端侧的头号可变参数。

## 演进与未来方向（判断）

以下为判断，锚定 2025-2026 真实工作。查证日期：2026-06-07。

第一，**整个领域在拼命砍步数，把 diffusion world model 从"多步"逼向"少步乃至一步"**。consistency model、distillation、尤其 flow matching 是主线——GAIA-2（arXiv:2503.20523）用 flow matching 训练 latent diffusion world model 提升时间一致与采样效率，正是这个方向的产业落地。我的判断是：端侧 diffusion world model 的未来是个位数步，一旦步数降到个位，三重乘法里最大的放大因子被拆掉，diffusion 的端侧成本从"重复 forward"塌回"接近一次生成"，硬件需求从 latency-bound 回到单次网络的 compute-bound。评估端侧可行性时，sampling steps 是头号变量，且 2-3 年内会持续下探，不能用今天的 50 步判死刑。

第二，对架构师更关键：**diffusion world model 的 denoiser 主干正全面转向 DiT/Transformer，让它的单步 workload 和本 wiki 其他 Transformer 篇收敛**——同样 GEMM + attention + norm，同样吃 attention 的 `N²` 与 activation memory。好处是硬件不必为 diffusion 单独优化一套算子，坏处是视频/3D 的 token 数把 attention 成本顶起来。对 archax，diffusion world model 应建模为"可参数化 denoiser（DiT/U-Net）× 可变采样步数 × candidate × horizon"的复合工作点，其中采样步数是 Interaction 轴上必须显式扫描的迭代维度；它的步间依赖与 [latent rollout 的状态递推](latent-world-model.md)、[occupancy 自回归](occupancy-world-model.md) 同属带状态的迭代推理主线，但 diffusion 的迭代是"对同一帧反复精修"，而 rollout 的迭代是"沿时间推进"，两类迭代在 archax 里应区分建模。云端那支连 [Cloud Inference and Simulation Chip](../06-chip-workload-analysis/cloud-inference-and-simulation-chip.md)，端侧低步数那支连 [World Model Workload](../06-chip-workload-analysis/world-model-workload.md)。

## Workload Characterization

计算密度：denoising backbone（DiT/U-Net）反复执行，总计算随采样步数 × candidate × horizon 近线性放大；低步数方案显著改变端侧可行性，是 diffusion 区别于一次 forward 的核心。

访存模式：每步读写 latent / condition / KV / activation；高分辨率 video 或 3D occupancy 显著放大容量；classifier-free guidance 引入条件/无条件双分支，增加访存与计算。

并行性：candidate / sample 可并行，单步内部可并行；denoise step 间顺序依赖，不能跨步并行——这是 diffusion 并行性的硬上限。

数据复用：condition encoding（history/map/action）可在所有 denoise step 与 sample 间复用（一次算好缓存），是首要优化点；latent state 每步更新需高效驻留。

量化敏感度：denoiser 的 Conv/GEMM 可低比特，但多步误差累积放大，低比特需验证整条采样链；最后几步、边界细节、collision/contact 相关输出更敏感。

瓶颈类型：云端高分辨率生成 compute/capacity-bound；端侧 action/latent diffusion 是 latency-bound（步数 × 控制频率）；video/occupancy diffusion 还 memory-capacity-bound。

对硬件的核心需求：高效 repeated inference、condition cache 跨步复用、多 candidate/sample 并发、低步数（consistency/flow-matching）生成支持、低比特 denoiser、video/3D latent tiling——详见 [World Model Workload](../06-chip-workload-analysis/world-model-workload.md)。

## 参考来源

- Ho et al., `Denoising Diffusion Probabilistic Models`, NeurIPS 2020 / arXiv:2006.11239。成熟度：已落地，DDPM 出处。
- Rombach et al., `High-Resolution Image Synthesis with Latent Diffusion Models`, CVPR 2022 / arXiv:2112.10752。成熟度：已落地，latent diffusion 基础。
- Chi et al., `Diffusion Policy: Visuomotor Policy Learning via Action Diffusion`, RSS 2023 / arXiv:2303.04137。成熟度：已落地研究，多模态动作生成，端侧需严控步数。
- Russell et al. (Wayve), `GAIA-2: A Controllable Multi-View Generative World Model for Autonomous Driving`, 2025 / arXiv:2503.20523。成熟度：产业研究，flow matching 训练的 latent diffusion world model，查证日期：2026-06-07。
- NVIDIA, `Cosmos World Foundation Model Platform for Physical AI`, 2025 / arXiv:2501.03575。成熟度：2025 平台，diffusion/AR 两类 world foundation model，查证日期：2026-06-07。
</content>
