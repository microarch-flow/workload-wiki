# Edge-cloud Collaborative World Model

上级：[World Model and Generative Intelligence](README.md)
相关：[Cloud Inference and Simulation Chip](../06-chip-workload-analysis/cloud-inference-and-simulation-chip.md), [World Model Workload](../06-chip-workload-analysis/world-model-workload.md)

## 这页在回答什么问题

这页回答 World Model 为什么天然需要端云协同。最重的生成式训练、仿真扩增、长尾场景挖掘通常在云端完成；端侧需要的是低延迟、可验证、与 policy/planner 紧密结合的轻量预测。

## 职责拆分

| 位置 | 适合任务 | 原因 |
| --- | --- | --- |
| 云端训练 | 大规模 world model 训练、视频生成、仿真、数据闭环 | 数据和算力集中，允许高延迟 |
| 云端评测 | 多版本模型回放、scenario mining、counterfactual generation | 可大规模并行 |
| 端侧推理 | 短 horizon risk prediction、trajectory candidate scoring、fallback check | 必须低延迟、确定性强 |
| 端侧日志 | 失败触发、rare event capture、feature/log upload | 支撑闭环数据回流 |

端云协同的关键不是把同一个大模型拆成两半，而是把不同时间尺度的任务分给不同位置。云端负责扩展经验，端侧负责实时决策。

## 为什么不能全放云端

自动驾驶和机器人都不能依赖云端实时闭环控制。网络延迟、连接可靠性、隐私和安全要求决定了端侧必须能独立运行。云端 world model 可以生成训练数据、评估策略和更新模型，但端侧执行必须在本地完成。

## 为什么不能全放端侧

高保真 video world model、长 horizon simulation、多样本 counterfactual generation 的成本过高。端侧芯片更适合短 horizon、结构化、低延迟 world model，例如 future occupancy risk、latent dynamics、candidate trajectory scoring。

## 一句话理解

World Model 的云端负责“扩展经验”，端侧负责“实时判断”；端云协同决定生成式智能能否进入真实自动驾驶和机器人系统。

## Workload Characterization

- 计算密度：云端是大模型训练/生成/仿真的高吞吐计算；端侧是低 batch、短 horizon、低延迟推理。
- 访存模式：云端需要高吞吐数据湖和大 activation；端侧需要历史 latent、BEV/occupancy cache 和 candidate buffer。
- 并行性：云端按 scenario/model/sample 大规模并行；端侧按 candidate/action/head 小规模并行。
- 数据复用：云端生成数据回流训练；端侧历史 encoding 复用于多个 risk/cost 计算。
- 量化敏感度：云端训练可混合精度；端侧安全相关输出需校验量化误差。
- 瓶颈类型：云端瓶颈是集群调度、IO、显存和生成吞吐；端侧瓶颈是 latency、功耗、容量和确定性。
- 对硬件的核心需求：云端需要训练/仿真吞吐和数据管线；端侧需要高效缓存、低延迟 rollout、与感知和 planning 共享中间表示。

## 参考来源

- Caesar et al., `nuPlan: A closed-loop ML-based planning benchmark for autonomous vehicles`, arXiv:2106.11810，https://arxiv.org/abs/2106.11810，成熟度：闭环评测基准，查证日期：2026-05-29。
- Dauner et al., `NAVSIM: Data-Driven Non-Reactive Autonomous Vehicle Simulation and Benchmarking`, NeurIPS 2024 / arXiv:2406.15349，https://arxiv.org/abs/2406.15349，成熟度：新兴评测基准，查证日期：2026-05-29。
- NVIDIA, `Cosmos World Foundation Models`, 2025，https://www.nvidia.com/en-us/ai/cosmos/，成熟度：云端 physical AI world model 平台，查证日期：2026-05-29。
- Waymo, `The Waymo World Model`, 2026-02-06，https://waymo.com/blog/2026/02/the-waymo-world-model-a-new-frontier-for-autonomous-driving-simulation，成熟度：2026 云端仿真前沿，查证日期：2026-05-29。
