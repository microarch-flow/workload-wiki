# Workload Lens

上级：[Overview](README.md)
相关：[Research Questions](research-questions.md), [Workload Analysis Methodology](../06-chip-workload-analysis/workload-analysis-methodology.md), [Workload Characterization Axes](../06-chip-workload-analysis/workload-characterization-axes.md)

## 这页在回答什么问题

这页确立贯穿整份 wiki 的视角：作为架构师，怎么从一个算法看出它对硬件提出什么需求。它是阅读 01-05 算法章节时应该一直戴着的"眼镜"，而 06 的方法论篇是把这副眼镜落成可操作流程的工程版。

## 为什么需要一个固定的视角

同一个算法，算法工程师关心精度和泛化，架构师必须关心别的东西：它每搬一字节数据能算多少、访问规不规则、能不能并行、数据能不能留在片上复用、哪些部分能降精度、最先撑爆哪类资源。如果没有一个固定视角，读算法论文很容易被网络结构图带走，记住一堆模块名却说不出它对芯片意味着什么。

这副眼镜的核心是一句话：**算法决定的不是 FLOPs 总量，而是数据怎么流动**。而 deterministic NPU 的成败，几乎总是由数据流动（访存、搬运、状态、同步）决定，而不是峰值算力。这也是 archax "数据搬运优先"原则的认知基础。

## 这副眼镜的五个反射动作

读到任何一个算法时，应该条件反射地问五个问题，它们对应后续所有 Workload Characterization 小节的骨架。

第一问，计算密度：这个算法的核心计算是大矩阵乘、卷积、迭代去噪还是状态递推，每搬一字节能摊出多少计算？答案决定它是 compute-bound 还是 memory-bound——而这两类对硬件的要求几乎相反。第二问，访存模式：数据是连续、strided、gather/scatter、稀疏还是状态访问？这决定 RAM 的 row locality、DMA 的形态和 NOC 的压力。第三问，并行性：哪些维度并行不破坏依赖，哪些维度并行会增加通信？端侧的难点常常不是没有并行性，而是没有能被 batch=1 硬件稳定吃满的并行性。第四问，数据复用：weight、activation、state、cache 能不能留在片上反复用？复用决定 on-chip buffer 的价值。第五问，量化敏感度：哪些张量能低比特、哪些必须高精度？这决定低比特能否真正改变瓶颈。

这五问的完整定义在 [Workload Characterization Axes](../06-chip-workload-analysis/workload-characterization-axes.md)，全 wiki 统一用它，保证任意两个 workload 可比较。

## 这副眼镜最容易纠正的四个直觉错误

FLOPs 高就难部署——错。standard conv 和 Transformer prefill 的 FLOPs 很高，但数据复用充分，能喂满阵列；depthwise conv 和 decode 的 FLOPs 低，反而 memory-bound 更难。

参数大就需要算力——错。对 batch=1 的 decode，参数大首先意味着每步要搬更多权重字节，瓶颈是带宽。

端到端模型是一个 workload——错。E2E 自动驾驶和机器人 VLA 是多段性质相反的 stage 串成的系统 workload，必须拆开看，否则无法解释瓶颈。

World Model 等于视频生成——错。决策级 World Model 关心可控的状态预测和闭环一致性，视频生成只优化像素逼真度，二者 workload 目标不同。

## 这副眼镜怎么连到硬件和 archax

01-05 章节只戴前半副眼镜（看出 workload 特征），不展开硬件建模，保持算法知识的独立性。到了 06，这副眼镜才接上硬件：访存模式接 RAM 的 row locality/bank parallelism、接 DMA 的 scatter-gather、接 NOC 的 multicast/reduction；规则矩阵计算接 CIM；端云边界接 PCIE/host。最终落到 archax 的四个抽象——用 Resource 描述有什么资源、Topology 描述怎么连、Interaction 描述数据怎么动、Capability 描述能执行什么。

## 一句话理解

Workload lens 就是把"这个算法精度高不高"换成"这个算法的数据怎么流、最先撑爆哪类资源、该怎么映射到 NPU"——这是这份 wiki 区别于普通算法笔记的根本所在。

## 怎么用这页

读 01-05 任意一篇算法文档时，先看正文理解计算结构，再用上面五个反射动作主动预判它的 workload，最后对照该文末尾的 Workload Characterization 检验自己的判断。读 06 时，再把判断接到具体硬件资源和 archax 扫描变量上。
