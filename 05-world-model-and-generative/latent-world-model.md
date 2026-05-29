# Latent World Model

上级：[World Model and Generative Intelligence](README.md)
相关：[World Model Fundamentals](world-model-fundamentals.md), [World Model Workload](../06-chip-workload-analysis/world-model-workload.md)

## 这页在回答什么问题

这页回答 Latent World Model 为什么是最适合 planning 和 policy training 的 World Model 形式之一。它不直接预测像素，而是预测压缩后的 latent state，让模型把计算资源集中在对决策有用的状态演化上。

## 基本结构

```text
observation
   ->
encoder to latent state
   ->
latent dynamics: z_t, action_t -> z_{t+1}
   ->
reward / value / decoder / policy
```

Dreamer 系列是典型路线：学习 latent dynamics，再在 imagined rollout 中训练 policy。对自动驾驶和机器人来说，latent 可以表示 scene belief、object relation、contact state、future risk 或任务进度。

## 为什么不是所有系统都用 latent

Latent World Model 的优势是高效：rollout 比视频生成便宜得多，也更容易并发评估多个 candidate action。问题是 latent 可解释性弱，如果 latent 丢失了小障碍、接触边界、交通灯状态，planner 很难发现。

因此工程系统常把 latent 与结构化输出结合：latent 负责长时序预测，BEV/occupancy/object state 负责安全和几何约束。

## 与 JEPA 的关系

JEPA 类方法也强调在 representation space 中预测未来，而不是重建像素。V-JEPA 2 等工作把自监督视频表征推向 embodied planning 和 robot fine-tuning，说明 latent prediction 正在从表征学习走向可行动的 world model。

## 一句话理解

Latent World Model 用压缩状态预测未来，牺牲部分可解释性换取 rollout 效率，是最适合多候选 planning 的 world model 表示之一。

## Workload Characterization

- 计算密度：encoder 可能较重，latent dynamics 通常比 video decoder 轻；多候选 rollout 会放大总计算。
- 访存模式：主要缓存 latent state、action candidate、history context 和 value/policy head；容量远低于 video latent。
- 并行性：candidate action、imagined trajectory、environment batch 可高度并行；单 trajectory 内部时间递推有依赖。
- 数据复用：同一 observation encoding 可用于多个 imagined rollout；latent state 可在 policy/value/world model 间共享。
- 量化敏感度：latent dynamics 可尝试低比特；value/risk/collision 相关 latent 维度的误差需要校验。
- 瓶颈类型：训练侧瓶颈是 rollout 数和反向传播显存；推理侧瓶颈是 candidate 数、horizon 和实时预算。
- 对硬件的核心需求：小张量高并发、状态缓存、多 rollout 调度、低延迟 recurrent/Transformer dynamics。

## 参考来源

- Ha and Schmidhuber, `World Models`, 2018，https://worldmodels.github.io/，成熟度：经典基础，查证日期：2026-05-29。
- Hafner et al., `DreamerV3`, arXiv:2301.04104，https://arxiv.org/abs/2301.04104，成熟度：成熟 latent dynamics 路线，查证日期：2026-05-29。
- Bardes et al., `V-JEPA 2: Self-Supervised Video Models Enable Understanding, Prediction and Planning`, arXiv:2506.09985，https://arxiv.org/abs/2506.09985，成熟度：2025 前沿研究，查证日期：2026-05-29。
- LeCun, `A Path Towards Autonomous Machine Intelligence`, 2022，https://openreview.net/forum?id=BZ5a1r-kVsf，成熟度：概念框架，查证日期：2026-05-29。
