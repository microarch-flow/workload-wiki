# World Model Fundamentals

上级：[World Model and Generative Intelligence](README.md)
相关：[World Model Workload](../06-chip-workload-analysis/world-model-workload.md), [Robot World Model](../04-robotics-and-vla/robot-world-model.md)

## 这页在回答什么问题

这页定义 World Model：一个模型如果能学习环境状态如何随时间和 action 演化，并能用于预测、规划、仿真或 policy 训练，才称得上 World Model。单纯生成一段视觉上合理的视频，不足以构成自动驾驶或机器人需要的 World Model。

## 基本形式

```text
history observation + state + action
   ->
state encoder
   ->
dynamics model
   ->
future latent / future video / future BEV / future occupancy
   ->
planner / policy / simulator / evaluator
```

World Model 的核心是 dynamics。它要回答“如果现在采取这个 action，世界接下来会怎样”。这与纯 perception 的差别在于，World Model 需要跨时间预测；与普通 generative model 的差别在于，它必须与 action、state 和 task objective 绑定。

## 分类

| 类型 | 表示 | 适合任务 |
| --- | --- | --- |
| latent world model | 压缩 latent state | RL、planning、policy training |
| video world model | future images/video | simulation、data generation、人类可视化 |
| BEV world model | bird's-eye-view feature/grid | 自动驾驶空间预测 |
| occupancy world model | 3D occupied/free/unknown | collision/risk/planning |
| object-centric world model | object state and relation | 机器人操作、物理推理 |

不同表示不是互斥关系。一个系统可以用 video model 生成数据，用 latent model 做 policy rollout，用 occupancy model 做安全约束。

## 成熟度判断

- 成熟：Dreamer 系列 latent dynamics、model-based RL、短时 horizon 的仿真预测。
- 发展中：自动驾驶 BEV/occupancy world model、机器人 implicit world model、action-conditioned video generation。
- 前沿：Cosmos、Genie 2、GAIA-2、V-JEPA 2 等把 world foundation model 推向物理 AI、交互仿真和 embodied planning，但离直接安全闭环部署仍有距离。

## 一句话理解

World Model 是“可预测 action 后果的环境模型”；它把 workload 从单次 perception 推向长时序状态缓存、多候选 rollout 和生成式训练。

## Workload Characterization

- 计算密度：取决于表示；latent dynamics 较轻，video/diffusion world model 很重，BEV/occupancy 介于两者之间。
- 访存模式：history、action candidate、future horizon、map/object state 和 latent/video cache 会显著增加容量需求。
- 并行性：不同场景、不同 action candidate、不同 sample 可并行；单条 rollout 内部存在时间依赖。
- 数据复用：同一历史编码可复用于多个 future rollout；同一 world model 可服务 simulation、planning、data generation。
- 量化敏感度：latent rollout 可尝试低比特；collision boundary、contact、occupancy edge 和长期一致性对误差敏感。
- 瓶颈类型：云端训练通常受显存和数据吞吐限制；端侧推理受 latency、candidate 数和 cache 容量限制。
- 对硬件的核心需求：长上下文缓存、多样本并发、生成模型加速、时序 rollout 调度、与 policy/planner 共享中间表征。

## 参考来源

- Ha and Schmidhuber, `World Models`, 2018，https://worldmodels.github.io/，成熟度：经典基础概念，查证日期：2026-05-29。
- Hafner et al., `Mastering Diverse Domains through World Models`, DreamerV3, arXiv:2301.04104，https://arxiv.org/abs/2301.04104，成熟度：成熟研究路线，查证日期：2026-05-29。
- LeCun, `A Path Towards Autonomous Machine Intelligence`, 2022，https://openreview.net/forum?id=BZ5a1r-kVsf，成熟度：概念框架，查证日期：2026-05-29。
- NVIDIA, `Cosmos World Foundation Model Platform for Physical AI`, 2025，https://www.nvidia.com/en-us/ai/cosmos/，成熟度：2025 产业平台，查证日期：2026-05-29。
