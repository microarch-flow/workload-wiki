# Representative Works Index

上级：[Overview](README.md)
相关：[Representative Papers](../07-reference/representative-papers.md), [Source Tracking](../07-reference/source-tracking.md), [Unified Relationship Map](unified-relationship-map.md)

## 这页在回答什么问题

这页是面向 workload 的代表工作入口索引：不是完整文献表，而是按"每个工作带来了什么 workload 启示"组织的导航。完整带 arXiv 编号的清单和成熟度/查证日期，见 07 的 [Representative Papers](../07-reference/representative-papers.md) 与 [Source Tracking](../07-reference/source-tracking.md)；这里的价值是把工作和它对芯片的含义连起来。

## 基础组件：定义了基本 workload 形态

Transformer（Attention Is All You Need, 2017）把计算从局部卷积推向内容相关的 token 交互，奠定了 prefill/decode 分裂的根源。FlashAttention（2022）证明 attention 的瓶颈常在 HBM IO 而非 MAC，确立了"算子要和 memory hierarchy 一起设计"的原则。ResNet/MobileNet/EfficientNet 定义了 standard vs depthwise 卷积的复用分化。Mamba（2023）和 Mamba-2（2024）把长序列压力从 N² attention 转向 selective scan 和 state。DDPM/Latent Diffusion 定义了"成本随采样步数放大"的迭代生成 workload。详见 [Foundation Model Components](../01-foundation-model-components/README.md)。

## 感知与 BEV/Occupancy：定义了不规则访存与容量压力

Lift-Splat-Shoot（2020）和 BEVFormer（2022）分别代表 scatter 和 gather 两种 view transform，是 BEV 不规则访存的源头。BEVFusion（2023）代表多传感器共享 BEV 表示。SurroundOcc/Occ3D/OccWorld 把感知推到 3D/4D 占用，定义了容量优先的 workload。详见 [BEV Perception](../02-vision-and-3d-perception/bev-perception.md) 与 [Occupancy Prediction](../02-vision-and-3d-perception/occupancy-prediction.md)。

## 自动驾驶 E2E：定义了系统级延迟与同步 workload

UniAD（CVPR 2023）确立 query-based、planning-oriented 的密集多任务 E2E。SparseDrive（2024）用稀疏表示把推理较 UniAD 快约 5 倍，说明稀疏化能改变 E2E 的算力/延迟画像。DiffusionDrive（2024）引入截断扩散的多模态轨迹。Waymo EMMA（2024）把 E2E 直接建在多模态大模型上，使瓶颈转向 VLM 推理。详见 [Autonomous Driving Algorithms](../03-autonomous-driving-algorithms/README.md)。

## 机器人 VLA：定义了大模型+实时控制叠加的 workload

RT-2（2023）开创"动作当 token 自回归输出"。OpenVLA（2024）是开源 7B baseline。FAST（2025）用 DCT+BPE 压缩 action chunk，让自回归 VLA 重新可行。π0/π0.5（2024-2025）和 GR00T N1（2025，双系统 VLM+diffusion action head）代表 flow/diffusion 连续动作路线。这些定义了视觉 token 爆炸、action 输出两条路线、action chunking 摊薄前向的 workload 特征。详见 [Robotics and VLA](../04-robotics-and-vla/README.md)。

## World Model：定义了 rollout 乘法成本

DreamerV3 代表轻量 latent world model。GAIA-1（离散 token 自回归）到 GAIA-2（2025，latent diffusion 多视角）代表驾驶视频世界模型的演进。NVIDIA Cosmos（2025）是云端物理 AI 世界模型平台。V-JEPA 2（2025）代表 LeCun 的非生成式 latent 预测路线。它们共同定义了 `候选 × 时域 × 单步` 的乘法成本和端云分裂。详见 [World Model and Generative](../05-world-model-and-generative/README.md)。

## 维护说明

这页只在某个工作改变了"对某类 workload 的理解"时才更新；纯粹的新 SOTA 数字进 07 的清单即可。前沿方向（VLA、World Model、E2E AD）的成熟度判断以 [Source Tracking](../07-reference/source-tracking.md) 为准，更新时两边保持一致。

## 一句话理解

这页把代表工作按"它教会我们关于 workload 的什么"来组织，是 07 完整文献表的 workload 视角入口。
