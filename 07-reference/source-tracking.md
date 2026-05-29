# Source Tracking

上级：[Reference](README.md)
相关：[Representative Papers](representative-papers.md), [Reading Roadmap by Goal](reading-roadmap-by-goal.md)

## 这页在回答什么问题

这页记录本轮重写中需要联网查证或后续持续跟踪的前沿方向。成熟基础算法不需要每次重查，但 VLA、World Model、E2E 自动驾驶、Action Tokenizer、robot foundation model 的演进很快，需要显式维护来源口径。

## 查证口径

| 字段 | 要求 |
| --- | --- |
| 名称 | 论文、系统或官方项目名称 |
| 年份 | 论文年份、官方发布时间或 arXiv 年份 |
| 出处 | arXiv 编号、会议、官方博客或项目页 |
| 成熟度 | 已验证落地、研究成熟、开源 baseline、论文阶段、产业前沿、概念框架 |
| 查证日期 | 本轮统一使用 2026-05-29 |

## 本轮重点查证项

| 方向 | 代表项 | 当前口径 |
| --- | --- | --- |
| AD VLM/VLA | DriveLM、DriveVLM、EMMA、AutoVLA | 研究和原型阶段，不能写成量产闭环 |
| AD World Model | GAIA-1、DriveWorld、DriveWAM、Waymo World Model | 云端仿真和前沿研究为主 |
| Robotics VLA | RT-2、OpenVLA、π0、π0.5、GR00T N1、SmolVLA | 从 VLA baseline 走向 action tokenizer 和 embodied generalist |
| Action Tokenizer | FAST、SmolVLA、RT-2 action tokens | 2025 前沿，重点是 token 长度、精度和 latency |
| Robot World Model | Diffusion Policy、FLARE | 发展中，实时闭环仍前沿 |
| General World Model | Cosmos、Genie 2、V-JEPA 2 | 物理 AI / embodied planning 前沿 |
| Simulation benchmark | nuPlan、NAVSIM、ReSim | 评测与闭环数据基础设施 |

## 逐项来源记录

| 名称 | 年份 | 出处 | 成熟度 | 查证日期 |
| --- | --- | --- | --- | --- |
| DriveLM | 2024 | arXiv:2312.14150 / ECCV 2024 | VLM 数据与评测研究成熟 | 2026-05-29 |
| DriveVLM | 2024 | arXiv:2402.12289 / CoRL 2024 | 研究原型 | 2026-05-29 |
| EMMA | 2024 | arXiv:2410.23262 | 前沿研究原型 | 2026-05-29 |
| AutoVLA | 2025 | arXiv:2506.13757 | AD VLA 前沿研究 | 2026-05-29 |
| GAIA-1 | 2023 | arXiv:2309.17080 | AD video world model 代表 | 2026-05-29 |
| DriveWorld | 2024 | arXiv:2405.04390 / CVPR 2024 | 研究成熟 | 2026-05-29 |
| DriveWAM | 2026 | arXiv:2605.28544 | 2026 前沿研究 | 2026-05-29 |
| Waymo World Model | 2026 | Waymo official blog, 2026-02-06 | 产业仿真前沿 | 2026-05-29 |
| RT-2 | 2023 | arXiv:2307.15818 / CoRL 2023 | VLA 范式代表 | 2026-05-29 |
| OpenVLA | 2024 | arXiv:2406.09246 | 开源 VLA baseline | 2026-05-29 |
| π0 | 2024 | arXiv:2410.24164 | VLA flow/action 前沿研究 | 2026-05-29 |
| π0.5 | 2025 | arXiv:2504.16054 | open-world generalization 前沿研究 | 2026-05-29 |
| GR00T N1 | 2025 | arXiv:2503.14734 / NVIDIA Research | 产业研究原型 | 2026-05-29 |
| FAST | 2025 | arXiv:2501.09747 | action tokenizer 前沿研究 | 2026-05-29 |
| SmolVLA | 2025 | arXiv:2506.01844 | 高效 VLA 前沿研究 | 2026-05-29 |
| FLARE | 2025 | arXiv:2505.15659 / CoRL 2025 | robot implicit world model 前沿 | 2026-05-29 |
| Cosmos | 2025 | NVIDIA official platform | physical AI world foundation model 产业平台 | 2026-05-29 |
| Genie 2 | 2024 | Google DeepMind official blog | world model demo 前沿 | 2026-05-29 |
| V-JEPA 2 | 2025 | arXiv:2506.09985 | self-supervised video planning 前沿 | 2026-05-29 |
| nuPlan | 2022 | arXiv:2106.11810 | 闭环规划基准 | 2026-05-29 |
| NAVSIM | 2024 | arXiv:2406.15349 / NeurIPS 2024 | 新兴仿真评测基准 | 2026-05-29 |
| ReSim | 2025 | arXiv:2506.09981 | reliable world simulation 前沿研究 | 2026-05-29 |

## 后续维护规则

更新 03-06 前沿内容时，必须同步更新 [Representative Papers](representative-papers.md)。如果新增条目属于 2025-2026 前沿方向，至少补齐名称、年份、出处、成熟度和查证日期。

如果一个系统来自官方博客而非论文，需要明确写“官方系统/平台/技术博客”，不要伪装成论文。若论文阶段模型没有真实部署证据，应写为“论文阶段”或“研究原型”。

## Workload Characterization

本页是来源维护记录，不代表单一 workload。它的 workload 价值在于保证前沿模型引用不会过时或混用成熟度，从而避免芯片需求判断被错误的算法假设带偏。
