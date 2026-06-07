# World Model Fundamentals

上级：[World Model and Generative Intelligence](README.md)
相关：[World Model Workload](../06-chip-workload-analysis/world-model-workload.md), [JEPA and Self-supervised](../01-foundation-model-components/jepa-and-self-supervised.md), [World Model for Autonomous Driving](../03-autonomous-driving-algorithms/world-model-for-ad.md), [Robot World Model](../04-robotics-and-vla/robot-world-model.md)

## 这页在回答什么问题

这页定义 World Model 到底是什么、分几类，以及为什么 action-conditioned 是它的灵魂而非可选项。重点是把"学习环境如何随 action 演化"这件事拆成三类可建模的 workload（generative sensor、latent dynamics、occupancy rollout），看清它们对硬件的落点为何相差一两个数量级，而不是去推导 dynamics 的数学。看懂这页，05 章后面七篇就都是它的展开。

## 为什么它有效：直觉与类比

World Model 的直觉是**给智能体一个脑内沙盘，让它行动前先在脑子里推演"如果我现在这么做，接下来会怎样"**。老司机并线前会预演"我加速插进去，后车会不会逼停"，在脑里跑几个可能的未来挑一个安全的再动手；这正是 World Model 要学的能力——给定历史和一个候选 action，预测世界将如何演变。对应到机制，就是学一个 dynamics 函数 `(state_t, action_t) -> state_{t+1}`，能被反复调用形成 rollout。

为什么"条件于 action"是灵魂而不是装饰：一个只会续写未来视频的模型，回答的是"世界自己会怎样"，而决策需要的是"在**我**这么做之后世界会怎样"。对应到机制，前者的输入里没有 action 这一路条件，无法对比"变道 vs 保持"两个分支的后果，于是再逼真也喂不进 planning。把 action 接进 dynamics 的条件输入，模型才能对多个候选动作各跑一条 rollout、按 collision/comfort/progress 比较——这是 World Model 区别于普通 generative model 的根本，也是它能服务规划的根本。

由此引出本章贯穿的判断：**World Model ≠ 视频生成**。如果模型只把未来画得逼真，对规划价值有限——画面像不等于碰撞风险可评估。要的是几何一致、动态一致、可控、可度量安全。这呼应 [JEPA](../01-foundation-model-components/jepa-and-self-supervised.md) 的核心思想：重要的是预测可决策的结构（谁会动、会不会撞），不是逐像素的逼真，这也是为什么端侧规划更可能用 latent 而非重像素生成（见 [World Model Is Not Video Generation](world-model-is-not-video-generation.md)）。

## 基本形式：dynamics 是核心，rollout 是计算骨架

```text
history observation + state + action
   -> state encoder
   -> dynamics: (state_t, action_t) -> state_{t+1}
   -> [反复迭代 H 步] -> future latent / video / BEV / occupancy
   -> planner / policy / simulator / evaluator
```

这里有个对硬件最关键的事实：World Model 的成本不是一次 forward，而是一个**乘法**——`candidate count × horizon × per-step cost`。感知模型一帧就是一次 forward；World Model 要对多个候选动作（planning 挑最优、安全评估覆盖多种情况），每个候选沿时间迭代 H 步预测未来。典型量级：端侧规划的 candidate 数 8 到 64、horizon 几步到几十步（OccWorld 用 2 秒历史推 3 秒未来，nuScenes 上即 4 帧推 6 帧）；如果 per-step 用 diffusion，单步里还要再乘几十次去噪迭代或上千个 token，乘法直接爆炸到只能放云端。这个乘法结构是 06 [World Model Workload](../06-chip-workload-analysis/world-model-workload.md) 的起点。

## 三类 World Model：同一思想，三种 workload 落点

| 类型 | 表示 / 输出 | 单步成本 | 适合任务 | 代表 |
| --- | --- | --- | --- | --- |
| latent dynamics | 压缩 latent state（几百到几千维） | 最轻，小网络递推 | RL、planning、policy training、短 horizon 风险评估 | Dreamer、V-JEPA 2 |
| occupancy / scene rollout | 3D occupied/free/unknown、actor motion | 中，受 voxel 容量主导 | collision check、闭环安全预测 | OccWorld |
| generative video / sensor | future camera/lidar 观测（T×H×W latent） | 最重，大生成模型 ×多步采样 | 仿真、数据生成、corner case 回放 | GAIA-1/2、Cosmos、Genie |

这三类不是互斥的工程选择，而是一条从"轻、可上车"到"重、偏云端"的连续谱。一个完整系统常同时用三者：云端 video model 造数据、端侧 latent model 做 policy rollout、occupancy model 提供几何安全约束。它们之所以重要，是因为表示的选择直接决定单步成本——latent 单步是个小 GEMM，video 单步是个跑几十步去噪的大 DiT，中间差一两个数量级。这就是为什么"World Model 的 workload"不能一概而论，必须先问它预测什么表示。

## 一句话理解

World Model 是"能预测 action 后果的环境模型"，靠 action-conditioned rollout 支撑"预演候选动作选最安全"；它的价值在几何/动态一致与可度量安全而非画面逼真，把 workload 从单次 perception 变成 `candidate × horizon × per-step` 的乘法，且 per-step 成本随表示（latent / occupancy / video）相差一两个数量级。

## 演进与未来方向（判断）

以下为判断，锚定 2025-2026 联网核实的真实工作。查证日期：2026-06-07。

演进脉络：2018 年 Ha & Schmidhuber 的 World Models 和 2023 年 DreamerV3 确立了 latent dynamics + 想象中训练 RL 的范式；2023-2024 GAIA-1、Drive-WM、OccWorld、Vista 把世界建模引入自动驾驶；2025-2026 出现两个并行突破——一是把世界模型当"基础模型"做（NVIDIA Cosmos，arXiv:2501.03575，开源 world foundation model 平台；DeepMind Genie 3，2025-08 发布，文本生成可实时交互世界，24fps、720p、一致性维持数分钟，2026-01 以 Project Genie 形式对 Ultra 用户开放），二是把 latent 预测真正接到机器人行动（V-JEPA 2，arXiv:2506.09985，1M+ 小时视频预训练，其 action-conditioned 变体 V-JEPA 2-AC 仅用 62 小时 Droid 机器人视频微调，配 CEM 采样 + receding horizon 做规划，官方称推理比 Cosmos 快约 30×）。

我的判断分两支且正在分道扬镳。其一，**生成式 world foundation model（Cosmos/Genie/GAIA 类）正成为云端的"数据与仿真工厂"**，成熟度最高，因为它不受车端实时约束。其二，**latent dynamics 这一支正在变成端侧可行的方向**——V-JEPA 2-AC 用极少交互数据 + 无像素 decoder 的 latent 预测做实机规划，且明确比像素生成式快一个量级，正是端侧需要的形态。对架构师，这个分叉直接对应两类几乎相反的硬件需求：云端那支是训练吞吐 + 显存容量 + 数据 IO，连 06 [Cloud Inference and Simulation Chip](../06-chip-workload-analysis/cloud-inference-and-simulation-chip.md)；端侧那支是带状态的 action-conditioned 时序 rollout，连 06 [World Model Workload](../06-chip-workload-analysis/world-model-workload.md)。后者和 [Mamba 的状态递推](../01-foundation-model-components/mamba-and-ssm.md)、[Diffusion 的多步迭代](../01-foundation-model-components/diffusion-models.md) 同属"带状态的迭代推理"这条贯穿全 wiki 的主线，rollout 的 horizon 步数与 candidate 数是 archax Interaction 轴上必须显式扫描的两个维度。

## Workload Characterization

计算密度：随表示跨越一两个数量级——latent dynamics 单步是几百维 state 的小 GEMM（端侧可高频 rollout），video generative 单步是跑几十步去噪的大 DiT（只能云端），occupancy 居中受 3D head FLOPs 主导；训练侧普遍远重于端侧推理。

访存模式：必缓存 history token、action candidates、future horizon、map/object state；latent 的 state cache 是低维小张量，video 的 T×H×W latent 是容量大头，二者访存压力差一两个数量级。

并行性：不同 candidate action、不同 future sample、不同 scene 之间可并行；单条 rollout 内部沿时间严格依赖（state_{t+1} 依赖 state_t），是并行的硬断点——这与 Diffusion 的步间依赖、Mamba 的状态递推同构。

数据复用：同一历史 scene encoding 复用于所有 candidate（`encode once → rollout many`，是 World Model 最重要的省算手段，避免乘法底数被无谓放大）；map/context token 跨 rollout 步复用。

量化敏感度：latent dynamics 可尝试低比特；collision boundary、occupancy edge、rare object、长期一致性对误差敏感，且误差沿 rollout 逐步累积，horizon 越长越敏感。

瓶颈类型：云端是训练吞吐、显存容量、数据 IO；端侧若做实时 rollout，则是 latency、state cache 容量、candidate 数与 horizon——两支瓶颈截然不同。

对硬件的核心需求：长上下文 token/latent 缓存、多 candidate/多 sample 并发、`encode once → rollout many` 的 condition 复用、带状态的迭代 state update、与 policy/planner 共享中间表征——云端与端侧是两套需求，详见 [World Model Workload](../06-chip-workload-analysis/world-model-workload.md)。

## 参考来源

- Ha and Schmidhuber, `World Models`, NeurIPS 2018 / arXiv:1803.10122。成熟度：经典基础概念，latent world model 起点。
- Hafner et al., `Mastering Diverse Domains through World Models (DreamerV3)`, arXiv:2301.04104。成熟度：成熟研究路线，latent dynamics + 想象中训练 RL。
- Assran et al., `V-JEPA 2: Self-Supervised Video Models Enable Understanding, Prediction and Planning`, 2025, arXiv:2506.09985。成熟度：研究系统，action-conditioned latent world model 用于机器人规划，查证日期：2026-06-07。
- NVIDIA, `Cosmos World Foundation Model Platform for Physical AI`, 2025, arXiv:2501.03575。成熟度：2025 开源产业平台，world foundation model，查证日期：2026-06-07。
- Google DeepMind, `Genie 3: A New Frontier for World Models`, 2025-08（Project Genie 2026-01 公开）。成熟度：前沿研究，实时交互生成世界，查证日期：2026-06-07。
</content>
</invoke>
