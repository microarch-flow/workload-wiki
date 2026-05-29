# World Model for Autonomous Driving

上级：[Autonomous Driving Algorithms](README.md)
相关：[Diffusion Models](../01-foundation-model-components/diffusion-models.md), [World Model Workload](../06-chip-workload-analysis/world-model-workload.md)

## 这页在回答什么问题

这页回答 World Model 在自动驾驶中到底建模什么，以及它与 perception、planning、simulation 的关系。这里的 World Model 不是单纯视频生成，而是学习可用于预测、评估和规划的未来世界状态。

## 三类 World Model

| 类型 | 输入 | 输出 | 用途 |
| --- | --- | --- | --- |
| latent dynamics | history BEV / token / state | future latent | planning、risk scoring |
| occupancy / scene rollout | BEV / occupancy / actor state | future occupancy / actor motion | collision check、closed-loop forecast |
| generative video / sensor model | video / action / map | future camera/lidar-like observation | simulation、data generation、corner case replay |

自动驾驶 World Model 的关键是 action-conditioned rollout：模型不只预测世界会怎样，还要预测在 ego 采取某个 action 后世界会怎样变化。

## 与 planning 的关系

World Model 可以服务两种规划方式：

- model-based planning：对多个 candidate trajectory 做 rollout，比较 collision、comfort、progress、rule cost。
- policy training：在生成或仿真的未来中训练 planning/action model，减少真实数据覆盖不足的问题。

如果 World Model 只是生成好看的视频，它对规划价值有限。对自动驾驶更重要的是几何一致性、动态一致性、可控性和可评估性。

## 2025-2026 观察

2023-2024 的 GAIA-1、Drive-WM、DriveWorld 等工作把生成式世界建模引入自动驾驶。到 2025-2026，趋势开始从“生成传感器观测”转向“可控、可评估、可与 planning/action 结合”的世界模型，例如 Wayve 的 GAIA-2（latent diffusion、多视角、结构化条件）和厂商公开的 world model simulation 路线。

成熟度上，World Model 已经适合做数据生成、场景理解预训练和仿真增强；直接承担车端闭环规划核心仍属于前沿。

## 难点

- 长时序一致性：几秒到十几秒 rollout 中，车道、障碍物和交通灯状态不能漂移。
- 多 agent 交互：其他交通参与者会响应 ego action。
- 可度量安全：生成质量不等于碰撞风险可验证。
- 延迟和成本：大规模生成模型训练昂贵，车端实时 rollout 更困难。

## 一句话理解

自动驾驶 World Model 的目标是让系统在行动前先模拟未来；它把 workload 从单帧感知扩展到 action-conditioned temporal rollout 和大规模仿真生成。

## Workload Characterization

- 计算密度：latent dynamics、diffusion/transformer rollout、video decoder 或 occupancy decoder 都可能很重；训练侧远重于车端推理。
- 访存模式：历史 token、future horizon、multi-agent state、map context 和 action candidates 需要大容量缓存。
- 并行性：不同 candidate action、不同 future sample、不同 scene 可并行；单条 rollout 内部存在时间依赖。
- 数据复用：同一历史 scene encoding 可复用于多个 action candidate；map/context token 可跨 rollout 复用。
- 量化敏感度：latent dynamics 可尝试低比特；长期一致性、collision boundary、rare object generation 对误差敏感。
- 瓶颈类型：云端是训练吞吐、显存容量、数据 IO；车端若做实时 rollout，则是 latency、cache 和 candidate 数量。
- 对硬件的核心需求：长上下文 token/latent 缓存、多样本并发、生成模型加速、trajectory candidate 批处理、仿真训练集群吞吐。

## 参考来源

- Hu et al., `GAIA-1: A Generative World Model for Autonomous Driving`, arXiv:2309.17080，https://arxiv.org/abs/2309.17080，成熟度：生成式 AD world model 早期代表，查证日期：2026-05-29。
- Min et al., `DriveWorld: 4D Pre-trained Scene Understanding via World Models for Autonomous Driving`, CVPR 2024 / arXiv:2405.04390，https://arxiv.org/abs/2405.04390，成熟度：研究成熟，查证日期：2026-05-29。
- Wang et al., `Driving into the Future: Multiview Visual Forecasting and Planning with World Model for Autonomous Driving (Drive-WM)`, CVPR 2024 / arXiv:2311.17918，https://arxiv.org/abs/2311.17918，成熟度：研究成熟，可与 E2E planning 结合，查证日期：2026-05-29。
- Russell et al., `GAIA-2: A Controllable Multi-View Generative World Model for Autonomous Driving (Wayve)`, arXiv:2503.20523，https://arxiv.org/abs/2503.20523，成熟度：2025 产业研究，latent diffusion 多视角世界模型，查证日期：2026-05-29。
- Waymo, `The Waymo World Model: A New Frontier For Autonomous Driving Simulation`, 2026-02-06，https://waymo.com/blog/2026/02/the-waymo-world-model-a-new-frontier-for-autonomous-driving-simulation，成熟度：产业前沿公开方向，查证日期：2026-05-29。
