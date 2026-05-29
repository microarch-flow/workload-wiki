# Workload Wiki

这份 wiki 用英文目录和文件名组织，但正文以中文为主；算法、模型、系统和硬件相关专有名词保留英文。

本 wiki 从 workload 视角组织：01-05 讲清算法的计算结构与设计动机，06 是绝对重心（把算法翻译成芯片架构 workload 画像，并连接 BUS/RAM/NOC/DMA/FAB/CIM/PCIE 七份硬件 wiki 与 archax），00 与 07 提供导航、术语、统一关系图、代表工作索引与全 workload 对比表。前沿方向（VLA、World Model、E2E 自动驾驶等）的内容经过联网查证，引用均带名称、年份与 arXiv 编号/会议。

## 写作原则

1. 01-05 章节负责讲清算法原理、计算结构、数据流和演进动机。
2. 每篇算法文档都必须包含 `Workload Characterization` 小节。
3. 06 是整份 wiki 的重心，负责把算法翻译为芯片架构 workload 画像。
4. VLA、World Model、E2E 自动驾驶、Action Tokenizer、GR00T、Mamba/SSM 前沿应用必须联网查证。
5. 06 章节需要显式连接 RAM、DMA、NOC、CIM、PCIE 等硬件 wiki，以及 archax 的 Resource/Topology/Interaction/Capability 抽象。

## 目录

- [00-overview](00-overview/README.md)
- [01-foundation-model-components](01-foundation-model-components/README.md)
- [02-vision-and-3d-perception](02-vision-and-3d-perception/README.md)
- [03-autonomous-driving-algorithms](03-autonomous-driving-algorithms/README.md)
- [04-robotics-and-vla](04-robotics-and-vla/README.md)
- [05-world-model-and-generative](05-world-model-and-generative/README.md)
- [06-chip-workload-analysis](06-chip-workload-analysis/README.md)
- [07-reference](07-reference/README.md)
