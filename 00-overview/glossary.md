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

online softmax：FlashAttention 的核心机制，维护滚动最大值和滚动归一化和，逐 K/V tile 增量修正输出，从不把 `N×N` 分数矩阵物化到 HBM。意味着 attention 可 streaming、把瓶颈从 HBM IO 拉回算力，应作为 NPU 的一等 datapath 原语而非拼凑 kernel（详见 [Attention Variants](../01-foundation-model-components/attention-variants-and-efficiency.md)）。

GQA / MQA：让多个 query head 共享一组 KV head 的结构性减 cache 手段（MQA 共享到一组，GQA 介于多头与单组之间）。直接压 decode 的 KV cache 容量与带宽，针对 memory-bound 的 decode 而非 prefill 算力。MLA（Multi-head Latent Attention）：把 KV 投影到低维 latent 再缓存（DeepSeek-V2），把 KV cache 结构性压一个数量级。意味着"KV 的有效比特数"正成为和参数量同级的部署指标——谁的 KV cache 小，谁能在同样 HBM 上服务更长上下文或更大 batch。

BEV（Bird's-Eye-View）：把多相机透视特征投影到 ego-centric 俯视栅格的表示。view transform / lift-splat：BEV 的投影步骤，scatter（lift-splat）或 gather（query 类 deformable attention），是 BEV 不规则访存的来源。LSS 的 forward scatter（每像素按预测深度把特征抛洒到 BEV 格子）与 BEVFormer 的 backward gather（每个 BEV 格子按几何反查去相机图采样）是方向相反的两种 view transform，对硬件都是几何驱动的不规则访存。Occupancy：把场景表示为 3D 体素的 occupied/free/unknown 或语义占用。

set prediction / NMS-free：DETR 类用固定数量 query + 匈牙利匹配直接输出对象集合，从机制上保证不重复，删掉 NMS。意味着检测/分割从"dense head + 手工后处理"变成"一组 query + set prediction"，去掉了 NMS 这类动态、需 CPU 介入的同步点，对 deterministic NPU 利好。mask classification：Mask2Former 类用"query embedding × pixel feature 点积得 mask"统一语义/实例/全景分割，把任务专用 head 收成同一套 query 读出；固定 query 数消除动态实例数，但高分辨率 mask 写回带宽不变。

E2E（端到端）：把感知-预测-规划串成一个可微网络（如 UniAD）。BC（Behavior Cloning）：用专家驾驶/操作示范模仿学习策略。covariate shift（协变量偏移）：BC 只在专家轨迹附近见过状态，一旦自身误差偏出专家分布就输出更离谱的动作、误差雪球式放大，是行为克隆的结构性缺陷；对治手段是闭环训练、轨迹扰动、RL 微调、World Model rollout。

VLA（Vision-Language-Action）：把动作纳入多模态大模型输出空间的机器人/驾驶范式。action tokenizer：把连续动作编码成 token 的方法——逐维 binning（RT-2/OpenVLA，token 数 × decode 延迟卡控制频率）或 FAST（对动作做 DCT-II 转频域 + BPE 压缩，把 token 数压一个量级让自回归 VLA 重新可行）。flow matching / diffusion action head：外挂一个 action expert 以 backbone 隐藏状态为条件，从噪声迭代去噪一次输出连续动作 chunk（如 π0、GR00T N1），瓶颈从"token 数"换成"去噪步数"。action chunk / action chunking：一次前向输出未来多步动作，用一次视觉编码覆盖多个控制周期、摊薄大模型前向；chunk 长度是"实时反应性 vs 推理频率"的旋钮。dual-system（慢-快双系统）：一个大 VLM（System 2）在低频做语义理解/规划，一个轻量 action policy（System 1）在高频做连续控制（如 Figure Helix、GR00T N1）；意味着一颗芯片要并发承载两个性格相反的工作点——memory-bandwidth 主导的大模型 decode 流和低延迟确定性的高频 action 流（详见 [VLA Fundamentals](../04-robotics-and-vla/vla-fundamentals.md)）。

World Model：给定状态和动作、预测未来状态的模型，强调可控状态预测和闭环一致性，区别于只优化像素逼真度的视频生成（详见 [World Model Is Not Video Generation](../05-world-model-and-generative/world-model-is-not-video-generation.md)）。action-conditioned rollout：以动作为条件、沿时间反复调用 dynamics 预测未来的过程，是 World Model 的灵魂；成本是 `candidate × horizon × per-step` 的乘法。latent world model：在压缩 latent 空间 rollout、不预测像素、无重 decoder、无多步采样的一支（Dreamer、V-JEPA 2-AC），单步成本比生成式低一两个数量级，是唯一能在端侧扛住 `candidate × horizon` 的表示。generative / video world model：预测未来像素观测（T×H×W latent + 像素 decoder + 多步采样）的一支（GAIA-2、Cosmos），最重，几乎是云端专属。rollout：沿时间反复调用 dynamics 预测未来的过程。JEPA：Joint-Embedding Predictive Architecture，在表征空间预测而非重建像素的自监督路线。

SSM / Mamba / state-space model：State Space Model，用一张固定大小、内容自适应的"便签"（状态）做状态递推建模长序列，避免 attention 的 `N²` 矩阵，状态大小恒定不随序列长度增长（相对 KV cache 的根本省法）；selective scan 是 Mamba 的核心算子，训练侧是高吞吐 associative scan（parallel prefix）、推理侧是低延迟逐步 state update。状态递推（state recurrence）：既非纯 GEMM、又非纯 attention 的第三类算子，是今天 NPU 的盲区，应在 archax Capability 轴预留。hybrid 架构：多数层用 Mamba/SSM 扛吞吐、少数层保留 attention 做精确检索的混层结构（Jamba、Nemotron-H），是长上下文的主流折中——纯 attention 超长上下文太贵、纯 SSM 精确召回太弱。Diffusion：通过多步去噪生成样本，成本随采样步数放大。

带状态的迭代推理：本 wiki 的一条贯穿主线——SSM 的状态递推、Diffusion 的多步去噪、World Model 的 action-conditioned rollout、video 流式状态，同属"state 常驻 + 逐步更新、单步轻但被迭代深度放大"的一类，是 archax Interaction 轴上的迭代维度（horizon/step/candidate），区别于一次性 forward。

## 硬件与 archax 术语

NPU：神经网络加速器，本 wiki 面向 deterministic NPU（强调 bounded latency、固定调度）。TOPS：每秒万亿次操作，峰值算力指标——本 wiki 反复强调它不足以判断可部署性。row locality / bank parallelism：DRAM 访问的局部性与并行性（详见 RAM wiki）。scatter-gather：按索引离散读写，DMA 的关键能力（详见 DMA wiki）。CIM：Compute-in-Memory，适合规则大块矩阵/卷积。

archax：本 wiki 服务的 AI 加速器架构探索工具链，用四个抽象描述系统——Resource（有什么资源：compute/SRAM/DRAM/DMA/NoC/host）、Topology（资源怎么连）、Interaction（数据怎么动：transfer/cache update/decode loop/rollout/sync）、Capability（能执行什么：Conv/GEMM/Attention/scatter-gather/sparse/quantization/stateful decode/SSM selective-scan 状态递推/hybrid attention-SSM）。"数据搬运优先"：archax 方法论原则，先看数据怎么流再看算力。

## 一句话理解

这页统一术语口径，核心是那组瓶颈标签和 archax 四抽象——它们是全 wiki 描述"算法→芯片需求"时反复使用的共同语言。
