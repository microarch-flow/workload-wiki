# Wiki Roadmap

上级：[Overview](README.md)
相关：[Research Questions](research-questions.md), [Reading Roadmap by Goal](../07-reference/reading-roadmap-by-goal.md), [Workload Analysis Methodology](../06-chip-workload-analysis/workload-analysis-methodology.md)

## 这页在回答什么问题

这页介绍各章节的职责，并按学习目标给出阅读路径。路径中的先后是建议，不是所有主题都必须满足的依赖。更细的现有路径见 [Reading Roadmap by Goal](../07-reference/reading-roadmap-by-goal.md)；其中的 workload 文章可按需阅读。

## 章节定位与依赖

| 章节 | 职责 | 建议的前置知识 |
| --- | --- | --- |
| [00 Overview](README.md) | 定位、导航、关系图和术语 | 无 |
| [01 Foundation Model Components](../01-foundation-model-components/README.md) | 模型组件、学习方法与生成方法 | 按具体文章补充基础 |
| [02 Vision and 3D Perception](../02-vision-and-3d-perception/README.md) | 感知任务、传感器和场景表示 | 按需参考 01 的组件 |
| [03 Autonomous Driving Algorithms](../03-autonomous-driving-algorithms/README.md) | 自动驾驶中的感知、规划与学习方法 | 按需参考 01 和 02 |
| [04 Robotics and VLA](../04-robotics-and-vla/README.md) | 机器人策略、动作表示和 VLA | 按需参考 01；感知相关内容可查 02 |
| [05 World Model and Generative](../05-world-model-and-generative/README.md) | 状态预测、生成方法及其应用 | 按主题参考 01，或结合 03、04 的场景 |
| [06 Chip Workload Analysis](../06-chip-workload-analysis/README.md) | 分析具体算法形成的工作负载及硬件映射 | 了解所选算法及其输入条件 |
| [07 Reference](../07-reference/README.md) | 论文、来源、索引和对比表 | 可随时查阅 |

    01 基础组件
      ├─ 按需参考 → 02 视觉与 3D 感知 → 03 自动驾驶
      ├─ 按需参考 → 04 机器人与 VLA
      └─ 按需参考 → 05 World Model

    02、03、04、05 中的具体算法 → 06 芯片工作负载分析（可选）
    00 提供导航；07 可在任意阶段查阅

图表示常见的参考关系，不表示每个专题都依赖图中的所有其他专题。World Model 可以在具备相关模型基础后单独阅读，也可以结合自动驾驶或机器人场景阅读。

## 推荐路径（按背景）

- **模型和算法入门**：从 [01 基础组件](../01-foundation-model-components/README.md) 选择需要的主题，再读 [02 视觉与 3D 感知](../02-vision-and-3d-perception/README.md) 中感兴趣的任务。遇到术语时查 [Glossary](glossary.md)。
- **自动驾驶**：先了解 [BEV 感知](../02-vision-and-3d-perception/bev-perception.md)，再进入 [03 自动驾驶算法](../03-autonomous-driving-algorithms/README.md)。BEV 是一种常用表示，不是所有自动驾驶方法的必经阶段。
- **机器人与 VLA**：从 [VLA Fundamentals](../04-robotics-and-vla/vla-fundamentals.md) 和 [Action Tokenizer](../04-robotics-and-vla/action-tokenizer.md) 开始，再按兴趣阅读 04 中的代表系统。
- **World Model**：先读 [World Model Fundamentals](../05-world-model-and-generative/world-model-fundamentals.md)，再选择 latent、video 或空间表示相关页面。需要时回到 01 补充模型基础。
- **芯片工作负载**：阅读 [Workload Lens](workload-lens.md)、[Workload Analysis Methodology](../06-chip-workload-analysis/workload-analysis-methodology.md) 和 [Workload Characterization Axes](../06-chip-workload-analysis/workload-characterization-axes.md)，然后结合具体算法选择 06 的文章。[AD and Robotics Chip Architecture Summary](../06-chip-workload-analysis/ad-robotics-chip-architecture-summary.md) 提供进一步的架构讨论入口。

## 重心提示

01–05 的目标是帮助读者理解算法与应用；06 面向希望研究芯片工作负载的读者。07 中的论文和来源记录可用于核对原始材料。部分现有文章仍以 workload 为中心，后续可逐步整理；阅读顺序也不代表算法的唯一演进路线。

## 一句话理解

先按问题选择算法主题，再按需要查阅代表工作或延伸到芯片工作负载。
