# World Model Is Not Video Generation

上级：[World Model and Generative Intelligence](README.md)
相关：[Video World Model](video-world-model.md), [World Model Workload](../06-chip-workload-analysis/world-model-workload.md)

## 这页在回答什么问题

这页澄清一个常见误解：World Model 不等于视频生成。视频生成可以是 World Model 的一种输出形式，但 World Model 的关键是可控预测、状态一致性和对 action 后果的建模。

## 区别

| 维度 | 视频生成 | World Model |
| --- | --- | --- |
| 目标 | 生成视觉上合理的视频 | 预测环境状态如何演化 |
| 条件 | prompt、首帧、文本、图像 | history state、action、map、task |
| 评价 | 画质、一致性、人类偏好 | planning value、安全、可控性、物理一致性 |
| 输出 | pixels/video latent | latent、object state、BEV、occupancy、video |
| 用途 | 内容生成、数据扩增 | simulation、planning、policy learning |

视频生成模型如果不能回答“ego 采取 action A 后是否会碰撞”，它对自动驾驶和机器人 planning 的价值有限。

## 为什么这个区分影响 workload

视频生成的 workload 常由 high-resolution latent diffusion、spatial-temporal attention 和 decoder 主导。World Model 的 workload 还要包括 action-conditioned rollout、多候选评估、状态缓存、风险/成本 head 和 closed-loop 调用。

因此，一个“看起来更大”的视频模型不一定是更好的 World Model。对架构探索来说，关键不是只看 FLOPs，而是看模型是否需要实时 rollout、是否要并发评估多个 action candidate、是否要输出结构化 state。

## 成熟度判断

视频生成在 2024-2026 进展很快，Sora、Veo、Genie、Cosmos 等推动长视频和物理场景生成。但 embodied AI 需要的可控 world model 仍处于发展中：画质已有明显进步，动作条件、交互一致性、可验证 safety metric 仍是硬问题。

## 一句话理解

视频生成强调“未来看起来像真的”，World Model 强调“未来状态能用于决策”；两者重叠但不能混同。

## Workload Characterization

- 计算密度：视频生成偏重 diffusion/transformer denoise 和 decoder；World Model 还叠加 action rollout 和 evaluator。
- 访存模式：视频模型有大 spatial-temporal latent；World Model 还需要 action/history/state cache。
- 并行性：视频 sample 可并行；World Model 的 action candidate 可并行，但 closed-loop 时间步有依赖。
- 数据复用：World Model 可复用历史 state encoding 评估多个 action；纯视频生成复用主要在 prompt/context。
- 量化敏感度：视频画质可容忍局部误差；World Model 的 collision/contact/occupancy 输出更敏感。
- 瓶颈类型：视频生成瓶颈是生成吞吐和显存；World Model 还受实时性和多候选 rollout 数量限制。
- 对硬件的核心需求：生成模型加速、长时序缓存、多候选并发、结构化输出 head、闭环调度能力。

## 参考来源

- OpenAI, `Video generation models as world simulators`, 2024，https://openai.com/index/video-generation-models-as-world-simulators/，成熟度：研究观察与系统展示，查证日期：2026-05-29。
- Google DeepMind, `Genie 2: A large-scale foundation world model`, 2024，https://deepmind.google/discover/blog/genie-2-a-large-scale-foundation-world-model/，成熟度：前沿 world model demo，查证日期：2026-05-29。
- NVIDIA, `Cosmos World Foundation Models`, 2025，https://www.nvidia.com/en-us/ai/cosmos/，成熟度：2025 产业平台，查证日期：2026-05-29。
- Waymo, `The Waymo World Model: A New Frontier For Autonomous Driving Simulation`, 2026-02-06，https://waymo.com/blog/2026/02/the-waymo-world-model-a-new-frontier-for-autonomous-driving-simulation，成熟度：2026 产业前沿，查证日期：2026-05-29。
