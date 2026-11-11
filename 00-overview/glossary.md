# Glossary

上级：[Overview](README.md)
相关：[Workload Characterization Axes](../06-chip-workload-analysis/workload-characterization-axes.md), [Workload Lens](workload-lens.md)

## 这页在回答什么问题

这页简要解释 wiki 中常见的算法、系统和芯片工作负载术语。各术语在具体论文或实现中的定义可能不同，应结合上下文和原始来源使用。

## 算法与系统范式

**Attention**：根据 query 与 key 的关系对 value 聚合的一类机制。权重可以随输入内容变化；这不意味着普通 dense attention 一定会动态选择要访问的 key 位置。

**prefill / decode**：自回归生成中，prefill 处理输入前缀，decode 根据已有上下文增量生成输出。具体执行方式随模型和实现变化。[Transformer Workload](../06-chip-workload-analysis/transformer-workload.md) 讨论相关工作负载。

**KV cache**：部分自回归模型保存先前计算的 key 和 value，以供后续步骤复用。其大小取决于模型、序列长度、批量、精度和缓存方式。

**online softmax**：分块处理输入时，逐步更新 softmax 所需的归一化量和输出的计算方法。[FlashAttention](https://arxiv.org/abs/2205.14135) 使用分块减少存储器层级之间的读写；实际性能取决于输入和实现。相关变体见 [Attention Variants](../01-foundation-model-components/attention-variants-and-efficiency.md)。

**GQA / MQA / MLA**：不同的 attention 或缓存结构设计。GQA 和 MQA 让多个 query head 共享 key/value head；MLA 使用 latent 表示组织缓存。缓存收益需要结合具体模型和实现计算。

**BEV（Bird's-Eye View）**：从俯视方向组织空间信息的表示。它可以来自不同传感器或融合过程。view transform 是将一种视角或坐标系下的特征转换到另一表示的过程；具体实现可能使用 scatter、gather 或其他方法。

**Occupancy**：描述空间中占用状态的表示或预测任务。不同方案可能使用体素、语义类别或其他空间表示；输出含义以具体定义为准。

**set prediction / NMS-free**：一些检测方法将输出建模为集合，并通过训练目标减少对非极大值抑制（NMS）的依赖。固定数量的 query 或匹配机制本身不保证预测结果绝无重复。**mask classification** 将分割结果与类别预测关联，具体输出形式取决于方法。

**E2E（端到端）**：强调跨多个处理阶段直接连接或联合优化的系统组织方式。具体边界、是否可微和训练过程需要按方案说明。**BC（Behavior Cloning，行为克隆）**：从示范数据学习策略的方法；当执行时遇到训练数据覆盖不足的状态，误差可能累积，具体影响取决于任务和训练方法。

**VLA（Vision-Language-Action）**：将视觉、语言与动作关联起来的一类模型或方法。**action tokenizer** 将动作编码为离散 token；**action chunking** 一次预测多个时间步的动作。部分方案使用 flow matching 或 diffusion 生成连续动作，也有方案将不同频率或职责的模块组合。详见 [VLA Fundamentals](../04-robotics-and-vla/vla-fundamentals.md)。

**World Model**：对环境状态或观测及其随时间变化进行建模的一类方法。输入、输出和用途因方案而异；并非所有 World Model 都以动作为条件。**rollout** 是沿时间重复预测未来的过程；**action-conditioned rollout** 还以动作序列为条件。**latent world model** 在潜在表示上建模动态；video world model 等方法可以生成或预测视频相关表示。两类不一定互斥，见 [World Model Is Not Video Generation](../05-world-model-and-generative/world-model-is-not-video-generation.md)。

**JEPA（Joint-Embedding Predictive Architecture）**：在表征空间中进行预测的一类方法。**SSM（State Space Model）**：使用状态描述序列变化的模型族；Mamba 是相关方法之一。**hybrid 架构**在一个模型中组合不同组件，例如 attention 和 SSM。**Diffusion** 通常通过多个步骤将噪声变换为样本；步骤数及成本取决于具体方法。

**带状态的迭代过程**：以某种状态为基础，按步骤更新或预测的过程。状态形式、步骤间依赖和计算成本随方法变化，不能仅据此认定不同模型具有相同硬件需求。

## Workload 刻画维度

**计算密度（Arithmetic Intensity）**：在指定计算和存储层级的统计边界下，操作量与数据传输字节数之比，常用 FLOP/Byte 表示。它可以帮助分析计算与带宽的关系，但瓶颈还取决于硬件能力、访问模式、并行度和实现，见 [Workload Characterization Axes](../06-chip-workload-analysis/workload-characterization-axes.md)。

**compute-bound / memory-bandwidth-bound**：分别指计算吞吐或数据传输带宽限制了某个特定工作负载的性能。**memory-capacity-bound** 描述可用存储容量成为约束的情况。**irregular-access-bound / synchronization-bound** 分别描述不规则访问或同步等待对性能形成主要限制的情况。这些标签需要针对具体系统验证。

**时延约束**：对一次请求或一段端到端执行允许的时间提出要求；它本身不说明造成延迟的原因。**数据复用**指多次使用已加载或已计算的数据；是否能在某一级存储中复用取决于容量和映射。**量化敏感度**描述精度变化对结果的影响，应结合张量、任务和评价指标分析。

## 硬件与 archax 术语

**NPU**：用于神经网络计算的处理器或加速器。**TOPS**：每秒万亿次操作，通常用于表达特定精度和条件下的吞吐能力；单独的峰值 TOPS 不能说明应用的实际性能。

**row locality / bank parallelism**：与 DRAM 行访问局部性及 bank 并行访问有关的概念。**scatter-gather**：对分散位置的数据进行读取或写入的访问方式。**CIM（Compute-in-Memory）**：在存储器内或附近执行部分计算的架构方向，具体能力和约束取决于实现。

**archax**：相关的架构探索项目。本仓库部分工作负载文章使用 Resource、Topology、Interaction 和 Capability 等术语；其精确定义和工具支持范围应以对应版本的项目文档为准。

## 一句话理解

先用术语描述算法和系统，再在具体执行条件下使用工作负载及硬件术语。
