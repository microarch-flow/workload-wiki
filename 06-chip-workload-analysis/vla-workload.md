# VLA Workload

上级：[Chip Workload Analysis](README.md)
相关：[Robotics and VLA](../04-robotics-and-vla/README.md), [Transformer Workload](transformer-workload.md), [VLA Fundamentals](../04-robotics-and-vla/vla-fundamentals.md)

## 这页在回答什么问题

这页分析 Vision-Language-Action 模型的芯片压力。VLA 的难点是它把三个本来分属不同 workload 的东西压在一颗端侧芯片上：一个几 B 参数的 VLM backbone（大模型推理）、爆炸式增长的 visual token（高带宽输入）、以及一个必须在几十毫秒内闭环的 action 输出（强实时控制）。它既不是普通 LLM 推理，也不是传统轻量机器人 policy，而是两者最难的部分的叠加。

## visual token 爆炸：VLA 比纯 LLM 更难的第一个原因

纯文本 LLM 的 prompt 是几百到几千个 token。VLA 的输入里，视觉占了大头：一张图经过 ViT 切 patch 后产生数百个 visual token，多相机（机器人常有主相机+腕部相机，自动驾驶有环视 6–8 路）会把 visual token 数量乘上相机数，再叠加历史帧就更多。这些 visual token 全部进入 attention 和 KV cache，于是 prefill 的计算量和 KV cache 的容量都被视觉输入显著放大——VLA 的 KV cache 压力往往主要来自视觉而非语言。这直接落到 [Transformer Workload](transformer-workload.md) 里 KV cache 的容量与带宽分析。

## action 输出的两条路线：自回归 vs 连续，对 workload 影响相反

VLA 怎么把内部表示变成连续控制动作，是过去两年范式演进最快的地方，而不同路线的 workload 特征截然不同。

第一条是**动作当文本 token 自回归输出**（RT-2 开创）。动作被离散化成 token，像生成文字一样逐个 decode。这条路线的 workload 就是标准 LLM decode：batch=1、memory-bandwidth-bound、逐 token 串行延迟。问题是机器人控制需要高频（几十到上百 Hz），而朴素的逐维度逐时刻离散化在高频灵巧动作上 token 数爆炸、解码太慢。FAST（arXiv:2501.09747）用 DCT + BPE 把 action chunk 压缩成少量 dense token，让自回归 VLA 重新可行，π0-FAST 借此把训练效率提高约 5× 并能扩展到上万小时数据。

第二条是**连续动作头**，用 flow matching 或 diffusion 直接生成 action chunk（π0、π0.5、GR00T N1 的 diffusion action head）。这条路线不再逐 token 解码，而是一次性（经过若干次去噪迭代）输出一整段未来动作（action chunk），从而支持高频连续控制。它的 workload 不是 KV-bandwidth-bound 的逐 token decode，而是一个固定迭代次数的小型生成循环——计算更规则，但去噪步数成为新的延迟因子。

GR00T N1（arXiv:2503.14734）的双系统结构很能说明 VLA 的 workload 分层：System 2 是慢速 VLM（2.2B，基于 Eagle VLM）做语义理解，System 1 是快速 diffusion action head 做高频动作生成——前者偏 compute/带宽，后者偏低延迟循环，两者对硬件需求不同，必须分别建模。

## action chunk 与控制频率：摊薄昂贵的大模型前向

VLA 的一个关键工程手段是 action chunking：一次大模型前向输出未来 H 步动作，机器人执行这一段时不必每步都重跑 VLM。这把昂贵的 visual encoding + VLM 前向的成本摊到 H 个控制周期上，是端侧能跑得动大 VLA 的前提。对芯片而言，这意味着 workload 呈现"低频重前向 + 高频轻执行"的节律，archax 建模时控制频率和 action chunk 长度共同决定了 VLM 前向的实际调用频率。

## 端侧 vs 训练侧：两个完全不同的 workload

端侧（机器人本体、车端）：batch=1、功耗固定、强实时。瓶颈是 memory capacity（放得下几 B 权重 + 视觉 KV cache）、KV cache bandwidth、以及每个 action 的 p99 latency。一个 7B VLM 即便 INT4 量化也要约 3.5 GB 权重常驻，加上视觉 KV cache，对端侧内存是实打实的压力。训练侧（云端）：瓶颈是多模态数据吞吐、跨 embodiment 数据对齐、大 batch 显存与通信——这是 [Cloud Inference and Simulation Chip](cloud-inference-and-simulation-chip.md) 的范畴，和端侧几乎无关。

## 可建模参数

`visual token count`（受相机数、分辨率、历史帧放大，是 attention 与 KV cache 的主驱动）、`VLM parameter size`（决定权重带宽与内存容量）、`action representation`（自回归 token vs flow/diffusion chunk，决定 decode 形态）、`action chunk length H` 与 `control frequency`（共同决定 VLM 前向的实际调用频率与实时预算）、`denoise steps`（连续动作头的迭代延迟）、`camera count`、`robot state dimension`。

## 硬件连接

RAM：VLM 权重、视觉主导的 KV cache、action history 共同占容量，是端侧内存规划的核心约束（见 RAM wiki）。DMA：相机输入、token buffer、KV cache 更新需要低延迟搬运（见 DMA wiki）。NOC：视觉 encoder、LLM block、action head 之间共享 token/state，需要清晰路径（见 NOC wiki）。CIM：GEMM/FFN/QKV projection 可用 CIM；action decode 的状态循环和安全控制不是 CIM 收益点。PCIE/host：低层控制器、传感器、安全监控若跨 host 会直接影响控制稳定性，必须近端化。

## archax 建模

Resource：VLM TOPS、SRAM/KV cache 容量、DRAM bandwidth、相机 ingress、controller interface。Topology：`visual encoder → token fusion → decode/diffusion loop → controller` 的低延迟路径，以及 System2(VLM)/System1(action head) 的资源划分。Interaction：prefill、decode 或 diffusion 去噪循环、action chunk rollout、robot state 更新、safety monitor 同步。Capability：GEMM/Attention、KV cache quantization、action tokenizer 或 flow/diffusion head、stateful decode、混合精度、bounded latency scheduling。archax 探索 VLA 时要把"自回归 token"和"连续 diffusion chunk"作为两个工作点分别评估，它们的瓶颈一个在 KV 带宽、一个在去噪循环延迟。

## 一句话理解

VLA 是把大 VLM 推理和高频实时控制压在同一颗端侧芯片上的 workload：视觉 token 撑大 KV cache，action 头的范式（自回归 token vs flow/diffusion chunk）决定 decode 瓶颈，action chunking 是让它在端侧跑得动的关键。

## Workload Characterization

- 计算密度：prefill 与视觉编码 compute-bound；自回归 action decode 是 KV-bandwidth/latency-bound；diffusion action head 是固定步数的小生成循环。
- 访存模式：权重和 visual token 连续；KV cache（视觉主导）和 action history 是状态访问；多相机放大输入带宽。
- 并行性：视觉编码、head、候选动作可并行；自回归 action token 串行；diffusion 去噪步串行但每步规则。
- 数据复用：instruction/context 在短任务内可复用；视觉 token 更新频繁；action chunk 摊薄 VLM 前向成本。
- 量化敏感度：VLM 权重可 4/8-bit（端侧必需）；action head、gripper/contact、state embedding、末端姿态回归需保守验证，因动作误差直接进入闭环。
- 瓶颈类型：端侧第一瓶颈是 memory capacity + per-action latency，叠加视觉 KV bandwidth；训练侧是多模态数据吞吐与模型规模。
- 对硬件的核心需求：低 batch VLM 推理、视觉 KV cache 管理、action decode/diffusion 低延迟、多传感器输入、控制器近端同步。

## 参考来源

- Zitkovich et al., `RT-2: Vision-Language-Action Models Transfer Web Knowledge to Robotic Control`, CoRL 2023 / arXiv:2307.15818。成熟度：已落地研究范式，动作当 token 的代表。
- Kim et al., `OpenVLA: An Open-Source Vision-Language-Action Model`, arXiv:2406.09246。成熟度：开源 VLA baseline。
- Black et al., `π0: A Vision-Language-Action Flow Model for General Robot Control`, arXiv:2410.24164。成熟度：2024 前沿研究，flow-matching 连续动作代表。
- Physical Intelligence, `π0.5: a Vision-Language-Action Model with Open-World Generalization`, arXiv:2504.16054。成熟度：2025 前沿研究。
- Pertsch et al., `FAST: Efficient Action Tokenization for Vision-Language-Action Models`, arXiv:2501.09747。成熟度：2025 前沿研究，DCT+BPE 动作 tokenizer。
- NVIDIA, `GR00T N1: An Open Foundation Model for Generalist Humanoid Robots`, arXiv:2503.14734。成熟度：2025 产业研究原型，双系统 VLM+diffusion action head。
