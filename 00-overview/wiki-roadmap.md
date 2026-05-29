# Wiki Roadmap

上级：[Overview](README.md)
相关：[Research Questions](research-questions.md), [Reading Roadmap by Goal](../07-reference/reading-roadmap-by-goal.md), [Workload Analysis Methodology](../06-chip-workload-analysis/workload-analysis-methodology.md)

## 这页在回答什么问题

这页给出整份 wiki 的章节依赖和推荐学习路径。它解释为什么章节是这个顺序、哪些章是基础、哪些是重心，以及不同背景的读者该怎么走最短路径。按目标的细化阅读链路在 [Reading Roadmap by Goal](../07-reference/reading-roadmap-by-goal.md)。

## 章节定位与依赖

00 是入口：建立 workload 视角、核心问题、统一关系图和术语。01 是基础组件层，是后续一切的算子底座。02 把组件组装成视觉与 3D 感知，并在末尾用 BEV 搭起通向自动驾驶的桥。03 和 04 分别是自动驾驶和机器人 VLA 两条应用线，05 是横跨两者的 World Model。06 是绝对重心，把前面所有算法翻译成芯片 workload，并连接七份硬件 wiki 和 archax。07 是索引层，提供多维索引、代表论文、对比表和来源跟踪。

```text
00 overview (视角/问题/关系图/术语)
   |
01 foundation components (算子底座)
   |
02 vision & 3D perception ---- BEV 桥 ----> 03 autonomous driving
   |                                              |
   |                                         04 robotics & VLA
   |                                              |
   +----------------> 05 world model <-----------+
                          |
06 chip workload analysis (重心: 算法 -> 芯片 workload -> 硬件 wiki + archax)
                          |
07 reference (索引/论文/对比表/来源)
```

依赖关系上，01 是 02-05 的前置；06 依赖 01-05 提供的算法理解（所以 06 的 workload 篇会反向链接算法篇）；但 06 的两篇方法论篇（[Workload Analysis Methodology](../06-chip-workload-analysis/workload-analysis-methodology.md) 和 [Workload Characterization Axes](../06-chip-workload-analysis/workload-characterization-axes.md)）确立的维度体系，反过来是 01-05 每篇 Workload Characterization 小节的统一标准。

## 推荐路径（按背景）

如果你是架构师、想最快建立 workload 分析能力：先读 00 的 [Workload Lens](workload-lens.md)，再直接进 06 的两篇方法论篇建立维度体系，然后回头按需查 01-05 的具体算法，最后读 06 的各 workload 篇和收尾的 [AD and Robotics Chip Architecture Summary](../06-chip-workload-analysis/ad-robotics-chip-architecture-summary.md)。这条路径把重心前置，适合已有硬件背景的人。

如果你算法基础较薄、想先补背景：按 01 → 02 → 03/04 → 05 顺序读，每篇先看正文理解计算结构，再看文末 Workload Characterization，建立"算法→workload"的条件反射，最后进 06 把这些 workload 接到硬件。

如果你只关心某条应用线（自动驾驶 / 机器人 / World Model）：用 [Reading Roadmap by Goal](../07-reference/reading-roadmap-by-goal.md) 里对应目标的最短链路，不必线性通读。

## 重心提示

这份 wiki 的价值密度集中在 06。01-05 是为 06 服务的背景：它们讲清算法为什么这样设计，但始终带着"作为 workload 意味着什么"的视角。如果时间有限，宁可少读几篇算法细节，也要把 06 的方法论篇、各 workload 篇和收尾篇读透——那里才是连接七份硬件 wiki 和 archax 的地方。

## 一句话理解

章节顺序是"组件→感知→应用→世界模型→芯片 workload→索引"，但价值重心在 06；架构师可以重心前置，算法补课者可以顺序通读，单线读者走按目标路径。
