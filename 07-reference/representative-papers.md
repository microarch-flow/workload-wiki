# Representative Papers

上级：[Reference](README.md)
相关：[Source Tracking](source-tracking.md), [Multi-dimensional Index](multi-dimensional-index.md)

## 这页在回答什么问题

这页列出本 wiki 反复引用的代表论文、模型和系统。它不是完整 bibliography，而是架构探索时最常用的锚点：每个条目都能对应到一个算法范式或 workload 特征。

## Foundation Components

| 方向 | 代表工作 | 成熟度 |
| --- | --- | --- |
| Transformer | `Attention Is All You Need`, NeurIPS 2017 | 经典基础 |
| FlashAttention | `FlashAttention`, NeurIPS 2022 | 成熟高效 attention 路线 |
| CNN | `ResNet`, CVPR 2016；`MobileNets`, arXiv:1704.04861；`EfficientNet`, ICML 2019 | 成熟 |
| Diffusion | `DDPM`, NeurIPS 2020；`Latent Diffusion`, CVPR 2022 | 成熟 |
| Mamba / SSM | `Mamba`, arXiv:2312.00752；`Mamba-2`, arXiv:2405.21060 | 发展中 |
| JEPA | `V-JEPA 2`, arXiv:2506.09985 | 2025 前沿 |

## Vision and 3D Perception

| 方向 | 代表工作 | 成熟度 |
| --- | --- | --- |
| Detection | YOLO、RetinaNet、FCOS、DETR | 成熟 |
| Segmentation | FCN、DeepLab、Mask R-CNN、Mask2Former | 成熟 |
| BEV | `Lift, Splat, Shoot`, arXiv:2008.05711；`BEVFormer`, arXiv:2203.17270；`BEVFusion`, arXiv:2205.13542 | 成熟研究 |
| Occupancy | `SurroundOcc`, arXiv:2303.09551；`OpenOccupancy`, arXiv:2303.03991；`OccWorld`, arXiv:2311.16038 | 发展中 |

## Autonomous Driving

| 方向 | 代表工作 | 成熟度 |
| --- | --- | --- |
| Planning-oriented E2E | `UniAD`, arXiv:2212.10156；`VAD`, arXiv:2303.12077 | 研究成熟 |
| Behavior Cloning | `PilotNet`, arXiv:1604.07316；`ChauffeurNet`, arXiv:1812.03079；`TCP`, arXiv:2206.08129 | 成熟 baseline |
| AD VLM/VLA | `DriveLM`, arXiv:2312.14150；`DriveVLM`, arXiv:2402.12289；`EMMA`, arXiv:2410.23262；`AutoVLA`, arXiv:2506.13757 | 2024-2025 前沿 |
| AD World Model | `GAIA-1`, arXiv:2309.17080；`GAIA-2`, arXiv:2503.20523；`Drive-WM`, arXiv:2311.17918；`Vista`, arXiv:2405.17398 | 前沿 |
| Benchmark / simulation | `nuPlan`, arXiv:2106.11810；`NAVSIM`, arXiv:2406.15349；`ReSim`, arXiv:2506.09981 | 发展中 |

## Robotics and VLA

| 方向 | 代表工作 | 成熟度 |
| --- | --- | --- |
| RT series | `RT-1`, arXiv:2212.06817；`RT-2`, arXiv:2307.15818；`Open X-Embodiment / RT-X`, arXiv:2310.08864；`RT-H`, arXiv:2403.01823 | 研究成熟 |
| Open VLA | `OpenVLA`, arXiv:2406.09246 | 开源 baseline |
| Action tokenizer | `FAST`, arXiv:2501.09747；`SmolVLA`, arXiv:2506.01844 | 2025 前沿 |
| Generalist robot model | `π0`, arXiv:2410.24164；`π0.5`, arXiv:2504.16054；`GR00T N1`, arXiv:2503.14734 | 2024-2025 前沿 |
| Robot world model | `Diffusion Policy`, arXiv:2303.04137；`FLARE`, arXiv:2505.15659 | 发展中 |

## World Model and Generative

| 方向 | 代表工作 | 成熟度 |
| --- | --- | --- |
| Latent World Model | `World Models`, 2018；`DreamerV3`, arXiv:2301.04104 | 成熟研究路线 |
| Video World Model | OpenAI Sora technical note, 2024；Google DeepMind `Genie 2`, 2024；NVIDIA `Cosmos`, 2025 | 前沿系统 |
| BEV / Occupancy World Model | `BEVWorld`, arXiv:2407.05679；`OccWorld`, arXiv:2311.16038 | 前沿 |
| Edge-cloud World Model | Waymo World Model, 2026；NVIDIA Cosmos, 2025 | 产业前沿 |

## Workload Characterization

本页是代表来源索引。条目选择标准是：能代表一个算法范式、能解释一个 workload 瓶颈，或属于 2025-2026 需要持续跟踪的前沿方向。
