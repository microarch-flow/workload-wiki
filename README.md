# Workload Wiki

这份 wiki 以 **AI 算法和模型**为主线，介绍它们要解决的问题、主要机制和应用方式。现有内容聚焦模型基础组件、视觉与 3D 感知、自动驾驶、机器人与 VLA，以及 World Model。

读者可以先从算法入手，理解输入、输出、数据表示和计算过程。对芯片架构感兴趣的读者，可以继续分析特定算法在给定条件下形成的工作负载。算法阅读不要求先完成硬件分析。

目录和文件名使用英文，正文以中文为主，专有名词按需要保留英文。

## 写作原则

1. 先说明算法解决的问题、输入和输出，再解释主要机制、适用条件和局限。
2. 根据主题介绍模型结构、训练与评估方法，并区分论文方案、实验结果和实际应用。
3. 用代表工作和交叉链接说明不同方法的联系。涉及演进或性能比较时，保留来源和适用条件。
4. 将芯片工作负载作为延伸：先明确输入规模和软件实现，再讨论数据流、硬件映射与可能的瓶颈。
5. 保留原有目录和链接。部分既有专题仍采用以 workload 为中心的写法，可在后续整理时逐步调整。

## 阅读路径

- **了解全貌**：从 [Overview](00-overview/README.md) 和 [Wiki Roadmap](00-overview/wiki-roadmap.md) 开始。
- **学习算法**：按需要阅读 01 的基础组件，再选择 02 的感知、03 的自动驾驶、04 的机器人，或 05 的 World Model。各路径不要求线性通读。
- **延伸到芯片**：先选定一个算法及其输入条件，再进入 [Chip Workload Analysis](06-chip-workload-analysis/README.md)。
- **查阅资料**：在 [Reference](07-reference/README.md) 查找论文、索引和来源记录。

## 目录

- [00-overview](00-overview/README.md)：定位、阅读路径、关系图和术语。
- [01-foundation-model-components](01-foundation-model-components/README.md)：模型与算法的基础组件。
- [02-vision-and-3d-perception](02-vision-and-3d-perception/README.md)：视觉、3D 感知及场景表示。
- [03-autonomous-driving-algorithms](03-autonomous-driving-algorithms/README.md)：自动驾驶算法与系统。
- [04-robotics-and-vla](04-robotics-and-vla/README.md)：机器人算法与 VLA。
- [05-world-model-and-generative](05-world-model-and-generative/README.md)：World Model 与生成式方法。
- [06-chip-workload-analysis](06-chip-workload-analysis/README.md)：算法到芯片工作负载的延伸分析。
- [07-reference](07-reference/README.md)：论文、索引、对比表与来源记录。
