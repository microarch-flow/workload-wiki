# Video World Model

上级：[World Model and Generative Intelligence](README.md)
相关：[World Model Is Not Video Generation](world-model-is-not-video-generation.md), [Diffusion for World Model](diffusion-for-world-model.md), [Cloud Inference and Simulation Chip](../06-chip-workload-analysis/cloud-inference-and-simulation-chip.md), [World Model Workload](../06-chip-workload-analysis/world-model-workload.md)

## 这页在回答什么问题

这页回答视频生成式 World Model 的价值、边界和为什么它几乎是云端专属负载。它把 workload 推到 `T×H×W latent × 多步采样 × candidate` 的乘积顶点，于是决定可部署性的不是单帧画质，而是这个四维乘积有多大——理解它，才知道为什么这一支留在云端做仿真/数据生成，而把端侧实时规划让给 latent/occupancy 表示。

## 为什么它有效：直觉与类比

视频 World Model 的直觉是**给系统配一个能按你的操作往下放的录像机**：喂进过去几帧加一个 action，它续拍出"接下来世界长什么样"的视频。它有效的根源是信息完整——一帧像素里同时编码了外观、遮挡、动态、光照、罕见物体，不用预先约定"该保留哪些状态量"，全在画面里。对应到机制，模型在像素或像素 latent 空间直接拟合时空联合分布 `p(future frames | history, action)`，所以理论上能表达任意复杂的场景演化，这是 latent/occupancy 这类预先压缩的表示换不来的表达力。

为什么实际都在 latent space 做而非 pixel space：直接在 512×512 像素上做多帧时空建模，张量规模和注意力成本爆炸。对应到机制，先用 video tokenizer / VAE 把像素压进低维 latent（如 GAIA-2 的做法），在 latent 上做 dynamics，最后才 decode 回像素——单步张量规模降一个量级，代价是引入 tokenizer/decoder 且压缩率影响保真。这与 [Diffusion Models](../01-foundation-model-components/diffusion-models.md) 里 latent diffusion 省算的逻辑同源。

代价藏在"视频"两个字里：它是 5D 张量（batch、time、height、width、channel），上面任何一维涨，成本成倍涨；再叠加 diffusion 的多步采样，就是把本已很重的单步又乘上几十次。能力来自信息完整，负担也来自信息完整。

## 计算结构与那个四维乘积

```text
history frames + optional action / text / map
   -> spatial-temporal tokenizer (压进 latent)
   -> latent dynamics: DiT / temporal attention / autoregressive rollout
   -> [diffusion 多步采样 or AR 逐 token]
   -> future video latent -> decoder to pixels / features
```

核心 workload 不是单帧渲染量，而是：

```text
total cost ≈ per-step denoiser × sampling steps × candidates × (T × H × W latent tokens)
```

量级感受：GAIA-2 生成至多 5 路、448×960 多视角时空一致视频，用 flow matching 训练以提升时间一致；一段几秒、多视角的 latent video 就是上万到几十万 token 量级，再乘采样步数和 candidate，单次生成的 compute 与 activation 容量直接把它锁在多卡云端。这与 06 [World Model Workload](../06-chip-workload-analysis/world-model-workload.md) 强调的乘法爆炸是同一件事，只是这里乘积的每一项都顶到最大。

两类时间建模各有性格。diffusion/DiT 式（GAIA-2、Cosmos）是"每步处理整段 latent video、步间顺序依赖"，硬件性格是重复 full forward；autoregressive 式（GAIA-1 的离散 token 续写、Genie 类）是"逐 token/逐帧生成、KV cache 增长"，硬件性格是 stateful decode。前者是宽而重复的计算问题，后者是窄而深的带宽问题，详见 [Diffusion for World Model](diffusion-for-world-model.md)。

## 价值与边界：信息完整 ≠ 可直接规划

价值清楚：信息完整让它天然适合仿真可视化、数据生成、corner case 回放、人类审查。GAIA-1/2、Waymo World Model（自动驾驶）和 NVIDIA Cosmos（physical AI）都靠这点做云端数据工厂——把长尾、counterfactual 场景可控地造出来喂回训练和闭环评估，这条线成熟度最高，因为不受车端实时约束。

边界同样清楚：视频不是直接可规划状态。对 planner，pixels 还要再被感知模型解释成 object/occupancy/risk/cost；而生成几何只要漂移几十厘米，人眼接受、collision metric 不接受。这正是"World Model ≠ 视频生成"在 workload 上的体现（见 [World Model Is Not Video Generation](world-model-is-not-video-generation.md)）——它逼真，但要为逼真付出像素 decoder + 多步采样的重成本，而端侧决策未必需要那些像素。

## 一句话理解

视频 World Model 用信息完整的像素续写提供最直观的未来模拟，但它把 workload 顶到 `T×H×W latent × 采样步数 × candidate` 的乘积极值，部署成本比 latent/BEV/occupancy 高一两个数量级，因而几乎是云端仿真与数据生成的专属负载。

## 演进与未来方向（判断）

以下为判断，锚定 2025-2026 联网核实的真实工作。查证日期：2026-06-07。

第一，**这一支的主战场已稳定在云端 world foundation model**。2025 的 NVIDIA Cosmos（arXiv:2501.03575）把视频 world model 做成开源基础模型平台 + video tokenizer，定位明确是 physical AI 的"数字孪生世界"用于训练 policy；DeepMind Genie 3（2025-08）做到文本生成可实时交互世界（24fps、720p、一致性数分钟）。我的判断是这条线会继续往"更长一致性、更强可控条件、更高分辨率"走，但始终是云端负载——实时交互的 Genie 3 也跑在云端而非端侧芯片。

第二，对架构师更关键的判断是**视频 World Model 正在和云端推理/仿真芯片的需求收敛**，而不是和端侧 NPU。它要的是高吞吐矩阵阵列、巨大 HBM 容量（装 T×H×W activation 与 KV cache）、video latent tiling、多 sample 并发、跨卡通信——这正是 06 [Cloud Inference and Simulation Chip](../06-chip-workload-analysis/cloud-inference-and-simulation-chip.md) 的画像。对 archax，视频 World Model 应建模为"大生成 denoiser（DiT/AR）× 采样步数 × candidate × 5D latent token"的云端工作点，其中采样步数与 latent token 数是把它压向云端的两个放大因子；端侧若想用它，唯一出路是先把表示从像素换成 latent/occupancy（即跳到本章其他篇），而不是优化视频生成本身。

## Workload Characterization

计算密度：DiT/temporal attention + diffusion denoise 计算量高，单步就是大模型 forward；分辨率、帧数、视角数、sample 数任一上涨都成倍放大，是本章最 compute-heavy 的表示。

访存模式：video latent 是 5D 张量（B×T×H×W×C），时间/空间/channel 同时扩张；AR 式还有随帧增长的 KV cache，activation 与 cache 容量是主压力，常 capacity-bound。

并行性：不同 sample、不同 candidate、不同 batch 可并行；单条 denoise chain 的步间、AR rollout 的帧间严格串行，是并行硬上限。

数据复用：history frame encoding 可复用于多个 future sample；map/action/text condition 跨 rollout 步与 sample 复用（一次算好缓存），是少数省算点。

量化敏感度：生成主干可混合精度；几何边界、交通灯、接触点、罕见小物体生成对误差敏感，且多步采样误差累积，低比特需验证整条采样链。

瓶颈类型：云端训练/生成受显存容量 + HBM 带宽 + 吞吐限制；端侧实时视频 world model 当前成本过高，基本不可行。

对硬件的核心需求：高吞吐 GEMM、大容量 activation/KV cache、video latent tiling、多 sample 并发、跨卡通信、生成模型调度——这套需求指向云端而非端侧，详见 [Cloud Inference and Simulation Chip](../06-chip-workload-analysis/cloud-inference-and-simulation-chip.md) 与 [World Model Workload](../06-chip-workload-analysis/world-model-workload.md)。

## 参考来源

- Hu et al., `GAIA-1: A Generative World Model for Autonomous Driving`, 2023, arXiv:2309.17080。成熟度：自动驾驶视频 world model 早期代表，离散 token 自回归。
- Russell et al. (Wayve), `GAIA-2: A Controllable Multi-View Generative World Model for Autonomous Driving`, 2025, arXiv:2503.20523。成熟度：产业研究，latent diffusion + flow matching，至多 5 路 448×960 多视角，查证日期：2026-06-07。
- NVIDIA, `Cosmos World Foundation Model Platform for Physical AI`, 2025, arXiv:2501.03575。成熟度：2025 开源产业平台，含 video tokenizer，查证日期：2026-06-07。
- Google DeepMind, `Genie 3: A New Frontier for World Models`, 2025-08。成熟度：前沿，实时交互视频生成 24fps/720p，查证日期：2026-06-07。
- Waymo, `The Waymo World Model: A New Frontier For Autonomous Driving Simulation`, 2026-02。成熟度：产业前沿，云端仿真方向，查证日期：2026-06-07。
</content>
