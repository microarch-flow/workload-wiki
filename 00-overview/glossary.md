# Glossary

上级：[Overview](README.md)
相关：[Workload Characterization Axes](../06-chip-workload-analysis/workload-characterization-axes.md), [Workload Lens](workload-lens.md)

## 这页在回答什么问题

这页是术语表，统一全 wiki 反复出现的概念口径，避免同一个词在不同页面被不同地用。重点收录 workload 刻画维度、算法范式和硬件/archax 术语；纯算法细节不在此展开，详见对应章节。

## Workload 刻画维度

计算密度（Arithmetic Intensity）：每搬运一字节数据能产生的有效计算量（FLOP/Byte）。它决定一个 workload 是 compute-bound（算力受限）还是 memory-bound（访存受限），是判断瓶颈的第一把尺子，定义见 [Workload Characterization Axes](../06-chip-workload-analysis/workload-characterization-axes.md)。

compute-bound：瓶颈是 MAC 阵列吞吐，典型如 standard conv、Transformer prefill。memory-bandwidth-bound：瓶颈是数据读写速度，典型如 Transformer decode、depthwise conv。memory-capacity-bound：瓶颈是状态/中间张量放不下，典型如长上下文 KV cache、3D occupancy。latency-bound：单次调用的实时预算最紧，典型如机器人 policy、batch=1 decode。irregular-access-bound：索引访问破坏 row/cache/DMA 效率，典型如 BEV view transform、稀疏 voxel。synchronization-bound：stage 间等待或 CPU/NPU 往返限制性能，典型如 E2E 的 safety shell。

数据复用（reuse）：weight/activation/state/cache 能否留在片上反复使用，决定 on-chip buffer 的价值。量化敏感度：某张量/算子能否低比特而不损精度，按 stage 和张量判断，不能笼统标"支持 INT8"。

## 算法与系统范式

prefill / decode：Transformer 推理的两个阶段。prefill 并行处理输入 prompt、compute-bound；decode 自回归逐 token 生成、memory-bandwidth-bound（详见 [Transformer Workload](../06-chip-workload-analysis/transformer-workload.md)）。KV cache：decode 时保存历史 token 的 Key/Value，随序列长度和 batch 线性增长，同时压容量和带宽。

BEV（Bird's-Eye-View）：把多相机透视特征投影到 ego-centric 俯视栅格的表示。view transform / lift-splat：BEV 的投影步骤，scatter（lift-splat）或 gather（query 类 deformable attention），是 BEV 不规则访存的来源。Occupancy：把场景表示为 3D 体素的 occupied/free/unknown 或语义占用。

E2E（端到端）：把感知-预测-规划串成一个可微网络（如 UniAD）。BC（Behavior Cloning）：用专家驾驶/操作示范模仿学习策略。

VLA（Vision-Language-Action）：把动作纳入多模态大模型输出空间的机器人/驾驶范式。action tokenizer：把连续动作离散化成 token 的方法（如 FAST 用 DCT+BPE）。action chunk：一次前向输出未来多步动作，用于摊薄大模型前向成本。flow matching / diffusion action head：用生成式方法直接输出连续动作 chunk 的路线（如 π0、GR00T N1）。

World Model：给定状态和动作、预测未来状态的模型，强调可控状态预测和闭环一致性，区别于只优化像素逼真度的视频生成（详见 [World Model Is Not Video Generation](../05-world-model-and-generative/world-model-is-not-video-generation.md)）。rollout：沿时间反复调用 dynamics 预测未来的过程。JEPA：Joint-Embedding Predictive Architecture，在表征空间预测而非重建像素的自监督路线。

SSM / Mamba：State Space Model，用压缩状态递推建模长序列，避免 attention 的 N² 矩阵；selective scan 是 Mamba 的核心算子。Diffusion：通过多步去噪生成样本，成本随采样步数放大。

## 硬件与 archax 术语

NPU：神经网络加速器，本 wiki 面向 deterministic NPU（强调 bounded latency、固定调度）。TOPS：每秒万亿次操作，峰值算力指标——本 wiki 反复强调它不足以判断可部署性。row locality / bank parallelism：DRAM 访问的局部性与并行性（详见 RAM wiki）。scatter-gather：按索引离散读写，DMA 的关键能力（详见 DMA wiki）。CIM：Compute-in-Memory，适合规则大块矩阵/卷积。

archax：本 wiki 服务的 AI 加速器架构探索工具链，用四个抽象描述系统——Resource（有什么资源：compute/SRAM/DRAM/DMA/NoC/host）、Topology（资源怎么连）、Interaction（数据怎么动：transfer/cache update/decode loop/rollout/sync）、Capability（能执行什么：Conv/GEMM/Attention/scatter-gather/sparse/quantization/stateful decode）。"数据搬运优先"：archax 方法论原则，先看数据怎么流再看算力。

## 一句话理解

这页统一术语口径，核心是那组瓶颈标签和 archax 四抽象——它们是全 wiki 描述"算法→芯片需求"时反复使用的共同语言。
