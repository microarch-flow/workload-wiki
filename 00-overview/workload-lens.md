# Workload Lens

上级：[Overview](README.md)
相关：[Research Questions](research-questions.md), [Workload Analysis Methodology](../06-chip-workload-analysis/workload-analysis-methodology.md), [Workload Characterization Axes](../06-chip-workload-analysis/workload-characterization-axes.md)

## 这页在回答什么问题

这页为希望研究芯片架构的读者提供一条可选路径：在理解算法之后，如何描述它在特定执行条件下形成的工作负载。完整的分析流程见 [Workload Analysis Methodology](../06-chip-workload-analysis/workload-analysis-methodology.md)。

## 为什么需要一个固定的视角

算法名称或参数量本身不足以确定硬件需求。分析时需要说明模型结构、输入规模、精度、软件实现、执行阶段和目标硬件。不同条件可能产生不同的数据流、资源需求和瓶颈。

可以按下面的顺序思考：

    算法与数据表示
        → 指定输入及软件执行方式
        → 描述计算、数据访问和依赖
        → 映射到指定硬件
        → 估算、建模或测量，并注明依据

这里的分析不能替代算法本身的学习，也不能把假设和实测结果混为一谈。

## 这副眼镜的五个反射动作

1. **计算**：有哪些运算？输入形状、精度、执行次数和阶段分别是什么？
2. **数据访问**：需要读取或写入哪些权重、中间结果和状态？访问模式是否依赖输入或索引？
3. **并行与依赖**：哪些计算可以并行？哪些步骤必须等待其他结果？并行会引入什么通信？
4. **数据复用**：哪些数据在给定映射和容量下可以复用？复用发生在哪一级存储？
5. **精度与约束**：不同精度对结果和资源有什么影响？目标场景有哪些延迟、吞吐、容量或功耗要求？

这些问题及其术语可参考 [Workload Characterization Axes](../06-chip-workload-analysis/workload-characterization-axes.md)。具体分析仍需写明测量边界和前提。

## 这副眼镜最容易纠正的四个直觉错误

- **只看 FLOPs 就能判断性能**：不能。执行时间还与数据移动、并行度、实现和硬件能力有关。
- **某种算法天然是某种瓶颈**：不能这样泛化。例如，同一 Transformer 的 prefill 和 decode 可能呈现不同特征，具体瓶颈仍需结合模型、批量、缓存和硬件验证。
- **端到端系统只能作为一个整体分析**：可以同时分析整体和内部阶段。拆分方式应与实际执行边界一致。
- **World Model 都有相同的执行成本**：不能。表示、预测长度、候选数量和实现方式都可能不同。

## 这副眼镜怎么连到硬件和 archax

分析具体实现时，可以进一步检查计算资源、存储层级、DMA、NoC 以及主机与设备之间的数据传输。端侧和云侧的约束也需要分别定义。

archax 的 Resource、Topology、Interaction 和 Capability 是相关架构讨论的入口。模型的确切语义和支持范围应以对应版本的项目文档为准；是否存在某类瓶颈，需要通过适当的估算、模型或测量验证。

## 一句话理解

Workload lens 是从具体算法和执行条件出发，研究计算、数据流与硬件资源关系的分析视角。

## 怎么用这页

先阅读感兴趣的算法文章，记下输入、输出和主要步骤。需要分析芯片时，再用本页的问题列出前提，选择 06 中对应的 workload 文章，最后明确哪些结论来自估算、模型或测量。
