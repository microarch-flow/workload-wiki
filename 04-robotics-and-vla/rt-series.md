# RT Series

上级：[Robotics and VLA](README.md)
相关：[VLA Fundamentals](vla-fundamentals.md), [OpenVLA](openvla.md), [VLA Workload](../06-chip-workload-analysis/vla-workload.md)

## 这页在回答什么问题

这页回答 Google DeepMind RT 系列为什么是机器人 VLA 的关键演进线。RT-1 证明大规模真实机器人数据可以训练通用 policy；RT-2 把 VLM 预训练知识接到动作输出；RT-X 把问题进一步推向跨机器人数据和跨 embodiment 泛化。

## RT-1

RT-1 的核心是把机器人控制建模为 Transformer policy：

```text
image history + natural language instruction
   ->
EfficientNet / tokenization
   ->
Transformer
   ->
discretized action
```

它的重要性在于规模化：使用大量真实机器人演示数据，让单一模型覆盖多任务操作。RT-1 的动作空间仍然主要服务特定机器人平台，但它展示了机器人 policy 可以从 task-specific model 走向大规模 multi-task model。

## RT-2

RT-2 的关键变化是把 web-scale VLM 引入机器人控制，并把 robot action 表示成 text-like tokens。这样模型既能做语言/视觉任务，也能输出机器人动作。

这一步的 workload 变化很直接：机器人控制不再只是一个轻量 policy network，而是 VLM backbone + action token decode。它换来了语义泛化能力，但也引入了更大的模型参数、KV cache 和 token decode latency。

## RT-X 和 Open X-Embodiment

RT-X 关注跨机器人数据。Open X-Embodiment 把多个机构、多个机器人形态、多个任务的数据标准化，目标是训练能跨 platform 迁移的 policy。对 workload 来说，cross-embodiment 不只是数据问题，也意味着输入状态、动作维度、相机视角和控制频率都更不统一。

## RT-H

RT-H 用 language motion 构建 action hierarchy，让高层 task language 和低层 action 之间多一层语义动作描述。它说明 VLA 并不一定要一步从 instruction 到 motor command，中间可以有可复用的动作层次。

## 一句话理解

RT 系列把机器人从单任务模仿学习推进到 VLA：先规模化机器人数据，再引入 web-scale VLM，最后尝试跨机器人和动作层次泛化。

## Workload Characterization

- 计算密度：RT-1 相对轻，RT-2/RT-X 引入更重的 VLM backbone；action head 通常小于视觉语言主干。
- 访存模式：image token、language token、history token、action token 和 robot state 需要联合缓存；跨机器人数据训练会增加输入格式处理成本。
- 并行性：训练可按 episode/task/robot 并行；推理侧视觉编码并行，action token decode 部分串行。
- 数据复用：VLM 预训练权重可复用 web knowledge；机器人 demonstration 复用到多个任务和机器人形态。
- 量化敏感度：视觉语言 backbone 可低比特；动作 token 边界、连续控制恢复和接触任务需更高精度验证。
- 瓶颈类型：RT-2 类模型车间/家庭部署的瓶颈是低 batch VLM latency；训练瓶颈是多源机器人数据 IO 与标准化。
- 对硬件的核心需求：VLM 推理、token decode、历史状态缓存、动作输出低延迟、多传感器输入预处理。

## 参考来源

- Brohan et al., `RT-1: Robotics Transformer for Real-World Control at Scale`, arXiv:2212.06817，https://arxiv.org/abs/2212.06817，成熟度：成熟研究代表，查证日期：2026-05-29。
- Zitkovich et al., `RT-2: Vision-Language-Action Models Transfer Web Knowledge to Robotic Control`, CoRL 2023 / arXiv:2307.15818，https://arxiv.org/abs/2307.15818，成熟度：VLA 代表，查证日期：2026-05-29。
- Padalkar et al., `Open X-Embodiment: Robotic Learning Datasets and RT-X Models`, ICRA 2024 / arXiv:2310.08864，https://arxiv.org/abs/2310.08864，成熟度：跨 embodiment 数据路线，查证日期：2026-05-29。
- Belkhale et al., `RT-H: Action Hierarchies Using Language`, RSS 2024 / arXiv:2403.01823，https://arxiv.org/abs/2403.01823，成熟度：动作层次研究，查证日期：2026-05-29。

## 旧版素材

- `/mnt/e/workload-wiki-old/04_机器人与VLA/RT系列.md`
