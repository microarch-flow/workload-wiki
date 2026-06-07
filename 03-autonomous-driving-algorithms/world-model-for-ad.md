# World Model for Autonomous Driving

上级：[Autonomous Driving Algorithms](README.md)
相关：[Diffusion Models](../01-foundation-model-components/diffusion-models.md), [World Model Workload](../06-chip-workload-analysis/world-model-workload.md), [JEPA and Self-supervised](../01-foundation-model-components/jepa-and-self-supervised.md)

## 这页在回答什么问题

这页回答 World Model 在自动驾驶中到底建模什么，以及它与感知、规划、仿真的关系。这里的 World Model 不是单纯视频生成，而是学习可用于预测、评估、规划的未来世界状态。这是 03 章需联网核实的前沿方向，内容反映 2025-2026 的当前状态。

## 为什么它有效：直觉与类比

World Model 的直觉是**给车一个脑内沙盘，让它行动前先在脑子里推演"如果我现在变道，接下来几秒会发生什么"**。人类老司机就是这么开的——并线前会预演"我加速插进去，后车会不会逼停"，在脑子里跑几个可能的未来，挑一个安全的再动手。World Model 就是把这种"先想后做"的能力学进模型：给定历史和一个候选动作，预测世界将如何演变。

最关键、也最容易被误解的一点是 **action-conditioned rollout**——模型不只预测"世界会怎样"，而是预测"在 ego 采取某个 action 后世界会怎样"。这个"条件于我的动作"是它区别于普通视频预测的灵魂，也是它能服务规划的根本：因为能预演"我这么开"的后果，才能对比多个候选动作选最安全的。

由此引出一个常被搞混的区分：**World Model ≠ 视频生成**。如果它只是生成一段好看逼真的未来视频，对规划价值有限——画面逼真不等于碰撞风险可评估。自动驾驶要的不是"画得像"，而是几何一致、动态一致、可控、可度量安全。这呼应 [JEPA](../01-foundation-model-components/jepa-and-self-supervised.md) 的核心思想——重要的是预测可决策的结构（谁会动、会不会撞），不是逐像素的逼真细节；也是为什么车端规划更可能用 latent world model 而非重像素生成（见 [World Model is not Video Generation](../05-world-model-and-generative/world-model-is-not-video-generation.md)）。

## 三类 World Model 与两种用法

| 类型 | 输入 | 输出 | 用途 |
| --- | --- | --- | --- |
| latent dynamics | history BEV/token/state | future latent | planning、risk scoring |
| occupancy/scene rollout | BEV/occupancy/actor state | future occupancy/actor motion | collision check、闭环预测 |
| generative video/sensor | video/action/map | future camera/lidar-like 观测 | 仿真、数据生成、corner case 回放 |

两种用法对应两种 workload。**model-based planning**：对多个候选轨迹做 rollout，比较 collision/comfort/progress/rule cost，选最优——这是脑内沙盘的直接用法。**policy training**：在生成或仿真的未来里训练 planning/action model，弥补真实数据覆盖不足——这恰好对治 [Behavior Cloning](behavior-cloning-e2e.md) 的 covariate shift（在仿真的偏离状态里学恢复）。

## 一句话理解

自动驾驶 World Model 让系统行动前先在脑内沙盘模拟未来，靠 action-conditioned rollout 支撑"预演候选动作选最安全"的规划；它的价值在几何/动态一致和可度量安全，而非画面逼真，把 workload 从单帧感知扩展到带状态的时序 rollout 和大规模仿真生成。

## 演进与未来方向（判断）

以下为判断，锚定 2025-2026 联网核实的真实工作。

演进脉络：2023-2024 的 GAIA-1、Drive-WM、DriveWorld、Vista 把生成式世界建模引入自动驾驶；到 2025-2026，重心从"生成传感器观测"转向"可控、可评估、可与 planning/action 结合"。代表是 Wayve 的 GAIA-2（latent diffusion，可生成至多 5 路、448×960 的时空一致多视角视频，条件于 ego 动力学、agent 配置、天气、道路语义等结构化输入）、Waymo 公开的 World Model 仿真路线，以及把世界模型从"预测"推向"规划"的工作（如 Policy World Model 的 state-action 协同预测）。

我的判断分两支，且这两支正在分道扬镳。其一，**生成式世界模型在云端仿真/数据生成上已接近实用**——GAIA-2、Waymo World Model 这类把长尾场景、corner case、counterfactual 用可控生成造出来，喂回训练和闭环评估（见 [Data Closed Loop and Simulation](data-closed-loop-and-simulation.md)），这条线成熟度最高，因为它不受车端实时约束。其二，**latent world model 用于车端规划仍属前沿**——要在车上实时跑 action-conditioned rollout、还要可验证安全，延迟和成本都没解决；近期更可能是"轻量 latent dynamics 做短 horizon 风险评估"而非完整生成式 rollout。换句话说，世界模型当前的主战场是云端的数据工厂，车端实时规划是下一个山头。

对架构师，这个分叉直接对应两类完全不同的硬件需求，是 06 [World Model Workload](../06-chip-workload-analysis/world-model-workload.md) 的核心。云端那支是**训练吞吐 + 显存容量 + 数据 IO + 生成模型加速**的大规模问题，连接 06 的 [Cloud Inference and Simulation Chip](../06-chip-workload-analysis/cloud-inference-and-simulation-chip.md)。车端那支是**带状态的 action-conditioned 时序 rollout**——同一历史 scene encoding 复用给多个候选 action（candidate 批并行）、latent state 跨 rollout 步缓存、单条 rollout 内沿时间串行，和 [Mamba 的状态递推](../01-foundation-model-components/mamba-and-ssm.md)、[Diffusion 的多步迭代](../01-foundation-model-components/diffusion-models.md)、[Video Understanding 的流式状态](../02-vision-and-3d-perception/video-understanding.md) 同属"带状态的迭代推理"这条贯穿全 wiki 的主线。对 archax，rollout 的 horizon 步数和 candidate 数是 Interaction 轴上必须显式扫描的两个维度，长时序一致性和 multi-agent 交互是建模的主要难点。

## Workload Characterization

计算密度：latent dynamics、diffusion/transformer rollout、video/occupancy decoder 都可能很重；训练侧远重于车端推理；生成式（视频）比 latent 重一个量级（多了像素 decoder + 多步采样）。

访存模式：历史 token、future horizon、multi-agent state、map context、action candidates 需大容量缓存；生成式还需反复读写高维 latent。

并行性：不同 candidate action、不同 future sample、不同 scene 可并行；单条 rollout 内部沿时间依赖，是并行断点。

数据复用：同一历史 scene encoding 复用于多个 action candidate（model-based planning 的关键收益）；map/context token 跨 rollout 复用。

量化敏感度：latent dynamics 可尝试低比特；长期一致性、collision boundary、rare object generation 对误差敏感（误差沿 rollout 累积）。

瓶颈类型：云端是训练吞吐、显存容量、数据 IO；车端若做实时 rollout，则是 latency、cache、candidate 数量与 horizon——两支瓶颈截然不同。

对硬件的核心需求：长上下文 token/latent 缓存、多样本/多 candidate 并发、生成模型加速、trajectory candidate 批处理、仿真训练集群吞吐——云端与车端是两套需求，详见 [World Model Workload](../06-chip-workload-analysis/world-model-workload.md)。

## 参考来源

- Hu et al., `GAIA-1: A Generative World Model for Autonomous Driving`, arXiv:2309.17080。成熟度：生成式 AD world model 早期代表。
- Gao et al., `Vista: A Generalizable Driving World Model with High Fidelity and Versatile Controllability`, NeurIPS 2024 / arXiv:2405.17398。成熟度：已落地研究，action-conditioned 可控生成。
- Wang et al., `Driving into the Future: Multiview Visual Forecasting and Planning with World Model (Drive-WM)`, CVPR 2024 / arXiv:2311.17918。成熟度：研究成熟，可与 E2E planning 结合。
- Russell et al. (Wayve), `GAIA-2: A Controllable Multi-View Generative World Model for Autonomous Driving`, arXiv:2503.20523。成熟度：2025 产业研究，latent diffusion 多视角，查证日期：2026-06-07。
- Waymo, `The Waymo World Model: A New Frontier For Autonomous Driving Simulation`, 2026-02。成熟度：产业前沿公开方向，查证日期：2026-06-07。
