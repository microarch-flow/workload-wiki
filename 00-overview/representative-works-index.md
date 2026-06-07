# Representative Works Index

上级：[Overview](README.md)
相关：[Representative Papers](../07-reference/representative-papers.md), [Source Tracking](../07-reference/source-tracking.md), [Unified Relationship Map](unified-relationship-map.md)

## 这页在回答什么问题

这页是面向 workload 的代表工作入口索引：不是完整文献表，而是按"每个工作带来了什么 workload 启示"组织的导航。完整带 arXiv 编号的清单和成熟度/查证日期，见 07 的 [Representative Papers](../07-reference/representative-papers.md) 与 [Source Tracking](../07-reference/source-tracking.md)；这里的价值是把工作和它对芯片的含义连起来。

## 基础组件：定义了基本 workload 形态

Transformer（Attention Is All You Need, 2017）把计算从局部卷积推向内容相关的 token 交互，奠定了 prefill/decode 分裂的根源。FlashAttention（2022）证明 attention 的瓶颈常在 HBM IO 而非 MAC，用 online softmax 让 attention 可 streaming、不物化 `N×N` 矩阵，确立了"算子要和 memory hierarchy 一起设计"的原则。GQA（2023）和 MLA（DeepSeek-V2, 2024）把 KV cache 从推理 trick 提升为模型结构性压缩，让"KV 的有效比特数"成为和参数量同级的部署指标。ResNet/MobileNet/EfficientNet 定义了 standard vs depthwise 卷积的复用分化。Mamba（2023）和 Mamba-2（2024）把长序列压力从 `N²` attention 转向 selective scan 和恒定大小 state；Jamba（2024）代表 attention+SSM 的 hybrid 混层——把"状态递推"引入算子集合，是今天 NPU 之外的第三类算子。DDPM/Latent Diffusion 定义了"成本随采样步数放大"的迭代生成 workload。详见 [Foundation Model Components](../01-foundation-model-components/README.md)。

## 感知与 BEV/Occupancy：定义了不规则访存与容量压力

Lift-Splat-Shoot（2020，forward scatter）和 BEVFormer（2022，backward gather）分别代表方向相反的两种 view transform，是 BEV 不规则访存的源头；SparseBEV（2023）代表向 sparse query-centric 演进、绕开稠密 frustum。DETR（2020）把检测从"dense head + NMS"变成"一组 query + set prediction"，Mask2Former（2022）用 mask classification 统一语义/实例/全景分割——这条 query 化主线把 NMS/动态实例数这类 CPU 同步点移出推理路径，对 deterministic NPU 利好。BEVFusion（2023）代表多传感器共享 BEV 表示。SurroundOcc/Occ3D/OccWorld 把感知推到 3D/4D 占用，定义了容量优先的 workload。详见 [BEV Perception](../02-vision-and-3d-perception/bev-perception.md) 与 [Occupancy Prediction](../02-vision-and-3d-perception/occupancy-prediction.md)。

## 自动驾驶 E2E：定义了系统级延迟与同步 workload

UniAD（CVPR 2023）确立 query-based、planning-oriented 的密集多任务 E2E。SparseDrive（2024）用稀疏表示把推理较 UniAD 快约 5 倍，说明稀疏化能改变 E2E 的算力/延迟画像。DiffusionDrive（2024）引入截断扩散的多模态轨迹。Waymo EMMA（2024）和 OpenDriveVLA（AAAI 2026）把 E2E 直接建在多模态大模型/VLA 上，使瓶颈转向 VLM 推理；行为克隆这条线还把 covariate shift（误差雪球）确立为闭环失稳的结构性根因，对治手段从 open-loop BC 转向闭环/RL/World Model 精炼。详见 [Autonomous Driving Algorithms](../03-autonomous-driving-algorithms/README.md)。

## 机器人 VLA：定义了大模型+实时控制叠加的 workload

RT-2（2023）开创"动作当 token 自回归输出"。OpenVLA（2024）是开源 7B baseline（7-DoF 各 256 bin）。FAST（2025）用 DCT-II+BPE 把动作 token 数压一个量级、让自回归 VLA 重新可行，并把 tokenizer 做成跨 embodiment 的标准接口。π0/π0.5（2024-2025）用 flow matching action expert 出连续动作 chunk；Figure Helix（2025）和 GR00T N1（2025）确立慢-快 dual-system——大 VLM（System 2，低频语义/规划）+ 轻量 action policy（System 1，高频连续控制）。这些定义了视觉 token 爆炸、action 输出两条路线（离散 token vs flow/diffusion）、action chunking 摊薄前向、以及一颗芯片并发承载两个相反工作点的 workload 特征。详见 [Robotics and VLA](../04-robotics-and-vla/README.md)。

## World Model：定义了 rollout 乘法成本

DreamerV3 代表轻量 latent world model。GAIA-1（离散 token 自回归）到 GAIA-2（2025，结构化条件 + 多视角一致的 latent diffusion）代表驾驶视频世界模型的演进。NVIDIA Cosmos（2025）是云端物理 AI 世界基础模型平台，把"先在数字孪生里训 policy"做成开源工厂。V-JEPA 2（2025）代表 LeCun 的非生成式 latent 预测路线，其 action-conditioned 变体 V-JEPA 2-AC 无像素 decoder、推理较 Cosmos 快约 30×，是端侧 latent world model 的代表。它们共同定义了 `候选 × 时域 × 单步` 的乘法成本和端云分裂——生成式（video）上云、latent 预测式上端，两类硬件需求几乎相反。latent rollout 的状态递推与 Mamba/SSM、Diffusion 多步同属"带状态的迭代推理"主线。详见 [World Model and Generative](../05-world-model-and-generative/README.md)。

## 维护说明

这页只在某个工作改变了"对某类 workload 的理解"时才更新；纯粹的新 SOTA 数字进 07 的清单即可。前沿方向（VLA、World Model、E2E AD）的成熟度判断以 [Source Tracking](../07-reference/source-tracking.md) 为准，更新时两边保持一致。

## 一句话理解

这页把代表工作按"它教会我们关于 workload 的什么"来组织，是 07 完整文献表的 workload 视角入口。
