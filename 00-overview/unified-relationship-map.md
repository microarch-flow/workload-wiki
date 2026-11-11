# Unified Relationship Map

上级：[Overview](README.md)
相关：[Wiki Roadmap](wiki-roadmap.md), [Robotics vs Autonomous Driving](../04-robotics-and-vla/robotics-vs-autonomous-driving.md), [World Model Fundamentals](../05-world-model-and-generative/world-model-fundamentals.md)

## 这页在回答什么问题

这页说明基础组件、感知任务、场景表示、学习方法和应用系统之间的联系。它们不属于同一个分类层次，也不构成必须依次经过的演进阶段。

## 共享的底座

[01 基础组件](../01-foundation-model-components/README.md) 介绍 CNN、Transformer、Diffusion、SSM 和自监督学习等主题。具体系统可以选用其中一种或多种方法，也可以采用本仓库未覆盖的方法。

组件可以跨任务复用，但相同组件的输入、模型规模、训练方式和部署条件可能不同。是否具有相似的工作负载，需要针对具体实现分析。

## 四条线如何分化又如何交汇

可以从几个相互交叉的维度理解 01–05：

| 维度 | 示例 | 回答的问题 |
| --- | --- | --- |
| 任务 | 检测、分割、轨迹预测 | 需要解决什么问题？ |
| 模型组件与方法 | CNN、Attention、Diffusion、SSM | 如何处理信息或生成结果？ |
| 场景表示 | BEV、occupancy、latent 表示 | 场景或状态以什么形式表达？ |
| 学习方法 | 行为克隆、自监督学习 | 从什么数据和目标中学习？ |
| 系统组织 | 模块化、端到端 | 各功能如何连接、训练或优化？ |
| 应用场景 | 自动驾驶、机器人 | 系统在什么环境中使用？ |

这些维度可以组合。例如，一个自动驾驶系统可以使用 BEV 表示，并采用模块化或端到端组织；使用 BEV 本身不决定系统是否端到端。端到端系统也可能保留可识别的任务模块。[UniAD](https://arxiv.org/abs/2212.10156) 是一个以规划为目标组织多个驾驶任务的例子。

## 自动驾驶与机器人 VLA 的同与异

两者都可能使用多模态输入、时序信息和动作预测。但传感器、动作空间、反馈方式和评价目标随具体系统变化，不能只按应用名称判断算法结构或实时要求。

自动驾驶与机器人之间的详细比较见 [Robotics vs Autonomous Driving](../04-robotics-and-vla/robotics-vs-autonomous-driving.md)。学习其中一个方向时，可以按需参考另一个方向，不要求先完整读完对方章节。

## World Model 的横向角色

World Model 关注对环境状态或观测随时间变化的建模。不同方案的表示、输入、输出和用途可能不同；部分方案以动作为条件，并用于规划或策略学习。

World Model 可以结合自动驾驶或机器人问题阅读，也可以作为模型方法单独研究。[World Model Fundamentals](../05-world-model-and-generative/world-model-fundamentals.md) 介绍基本概念；[World Model Is Not Video Generation](../05-world-model-and-generative/world-model-is-not-video-generation.md) 讨论它与视频生成的联系和区别。两者可以交叉，不能只凭是否生成视频判断一个模型能否用于决策。

## 横切四条线的三条主线

- **表示与输出**：同一任务可以使用不同表示和输出接口。比较时需要说明坐标系、时间范围和输出含义。
- **时间与学习**：时序建模、自监督学习和动作条件预测适用于不同问题。具体方法是否使用历史状态或动作，要看其模型定义。
- **执行与部署**：如果需要分析芯片工作负载，应先确定算法、输入规模和实现，再结合目标硬件检查计算、数据访问和延迟。可按场景参考 [Edge Inference Chip Requirements](../06-chip-workload-analysis/edge-inference-chip-requirements.md) 与 [Cloud Inference and Simulation Chip](../06-chip-workload-analysis/cloud-inference-and-simulation-chip.md)。

## 一句话理解

任务、组件、表示、学习方法和应用是可以交叉组合的维度；芯片工作负载分析是针对具体组合及其执行条件的延伸。
