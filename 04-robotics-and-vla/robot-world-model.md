# Robot World Model

上级：[Robotics and VLA](README.md)
相关：[World Model Fundamentals](../05-world-model-and-generative/world-model-fundamentals.md), [World Model Workload](../06-chip-workload-analysis/world-model-workload.md)

## 这页在回答什么问题

这页回答机器人 World Model 与 VLA 的关系。VLA 可以直接从 observation 到 action，但机器人任务包含接触、遮挡、物体动力学和长时序执行；World Model 的目标是在行动前预测未来状态，帮助 policy 评估动作后果。

## 建模对象

机器人 World Model 可以预测三类东西：

| 类型 | 输出 | 作用 |
| --- | --- | --- |
| latent state | future latent / belief state | 给 policy 提供可规划表示 |
| visual future | future image / video | 仿真、数据生成、失败回放 |
| physical outcome | object pose、contact、success/failure | 动作评估、风险判断 |

机器人场景与自动驾驶不同。自动驾驶更强调大范围交通参与者和几何空间；机器人更强调近场接触、可操作物体、手眼协调和动作导致的局部状态变化。

## 与 VLA 的结合

VLA 是 policy，World Model 是可用于预测的环境模型。两者可以三种方式结合：

- policy pretraining：用视频/交互数据学习可迁移的物理表征。
- action rollout：对候选动作预测未来，选择风险更低的动作。
- data generation：生成长尾交互数据，补充真实机器人演示不足。

常见误解：机器人 World Model 等于视频生成。实际上，漂亮视频不等于可控物理预测；对机器人更关键的是动作条件、接触结果、物体状态和可用于控制的 latent。

## 成熟度判断

Diffusion Policy 已经是成熟研究路线；基于视频或 latent 的机器人 World Model 正在从仿真和离线任务走向 policy 训练；直接实时参与真实机器人闭环决策仍是前沿。

## 一句话理解

Robot World Model 让机器人在执行前预测动作后果；它把 workload 从反应式 VLA 推向 action-conditioned rollout、生成式训练和多候选动作评估。

## Workload Characterization

- 计算密度：视频/latent world model 训练重，推理侧若做多候选 rollout 也会很重；Diffusion/Transformer 是常见主计算。
- 访存模式：历史观测、action candidate、object state、latent rollout 和未来帧缓存会放大容量需求。
- 并行性：不同 action candidate、sample、scene 可以并行；单条 rollout 内部受时间依赖限制。
- 数据复用：同一 observation encoding 可复用于多个动作模拟；同一生成数据可用于 policy training 和 failure replay。
- 量化敏感度：latent dynamics 可尝试低比特；contact boundary、collision、grasp success 等输出对误差敏感。
- 瓶颈类型：训练侧受数据/显存/生成模型吞吐限制；实时使用时受候选数、rollout horizon 和 latency 限制。
- 对硬件的核心需求：生成模型加速、多候选并发、历史 latent 缓存、仿真数据吞吐、与 policy decoder 共享表征。

## 参考来源

- Ha and Schmidhuber, `World Models`, 2018，https://worldmodels.github.io/，成熟度：经典基础概念，查证日期：2026-05-29。
- Hafner et al., `Mastering Diverse Domains through World Models`, DreamerV3, arXiv:2301.04104，https://arxiv.org/abs/2301.04104，成熟度：成熟研究路线，查证日期：2026-05-29。
- Chi et al., `Diffusion Policy`, RSS 2023 / arXiv:2303.04137，https://arxiv.org/abs/2303.04137，成熟度：机器人动作生成成熟研究，查证日期：2026-05-29。
- Zheng et al., `FLARE: Robot Learning with Implicit World Modeling`, CoRL 2025 / arXiv:2505.15659，https://arxiv.org/abs/2505.15659，成熟度：2025 前沿研究方向，查证日期：2026-05-29。

## 旧版素材

- `/mnt/e/workload-wiki-old/04_机器人与VLA/机器人World_Model.md`
