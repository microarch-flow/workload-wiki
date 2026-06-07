# Representative Papers

上级：[Reference](README.md)
相关：[Source Tracking](source-tracking.md), [Multi-dimensional Index](multi-dimensional-index.md)

## 这页在回答什么问题

这页列出本 wiki 反复引用的代表论文、模型和系统。它不是完整 bibliography，而是架构探索时最常用的锚点：每个条目都能对应到一个算法范式或 workload 特征。

## Foundation Components

| 方向 | 代表工作 | 成熟度 |
| --- | --- | --- |
| Transformer | `Attention Is All You Need`, NeurIPS 2017, arXiv:1706.03762 | 经典基础 |
| Normalization | `Pre-Norm (On Layer Normalization in the Transformer Architecture)`, ICML 2020, arXiv:2002.04745；`RMSNorm`, NeurIPS 2019, arXiv:1910.07467 | 成熟 |
| FlashAttention | `FlashAttention`, NeurIPS 2022, arXiv:2205.14135；`FlashAttention-2`, arXiv:2307.08691 | 成熟高效 attention 路线 |
| KV cache 压缩 | `GQA`, EMNLP 2023, arXiv:2305.13245；`MLA / DeepSeek-V2`, arXiv:2405.04434 | 已落地 |
| CNN | `ResNet`, CVPR 2016；`MobileNets`, arXiv:1704.04861；`EfficientNet`, ICML 2019；`ConvNeXt`, CVPR 2022, arXiv:2201.03545 | 成熟 |
| ViT token 治理 | `ViT`, ICLR 2021, arXiv:2010.11929；`NaViT (Patch n' Pack)`, NeurIPS 2023, arXiv:2307.06304；`DINOv2`, arXiv:2304.07193 | 成熟到前沿 |
| Diffusion | `DDPM`, NeurIPS 2020；`Latent Diffusion`, CVPR 2022；`Consistency Models`, ICML 2023, arXiv:2303.01469；`DiT`, ICCV 2023, arXiv:2212.09748 | 成熟 |
| Mamba / SSM / hybrid | `Mamba`, arXiv:2312.00752；`Mamba-2`, arXiv:2405.21060；`Jamba`, arXiv:2403.19887 | 发展中 |
| JEPA | `V-JEPA 2`, arXiv:2506.09985 | 2025 前沿 |

## Vision and 3D Perception

| 方向 | 代表工作 | 成熟度 |
| --- | --- | --- |
| Detection | `YOLO`, CVPR 2016；`Faster R-CNN`, arXiv:1506.01497；`FCOS`, ICCV 2019, arXiv:1904.01355；`DETR`（set prediction / NMS-free）, ECCV 2020, arXiv:2005.12872 | 成熟 |
| Segmentation | `FCN`, CVPR 2015, arXiv:1411.4038；`DeepLab`、`Mask R-CNN`；`Mask2Former`（mask classification）, CVPR 2022, arXiv:2112.01527；`SAM`, ICCV 2023, arXiv:2304.02643 | 成熟 |
| Video understanding | `C3D`, ICCV 2015, arXiv:1412.0767；`TimeSformer`, ICML 2021；`ViViT`, ICCV 2021；`VideoMAE`, NeurIPS 2022 | 成熟 |
| BEV | `Lift, Splat, Shoot`（LSS scatter）, arXiv:2008.05711；`BEVFormer`（gather）, arXiv:2203.17270；`BEVFusion`, arXiv:2205.13542；`SparseBEV`, ICCV 2023, arXiv:2308.09244 | 成熟研究 |
| Occupancy | `SurroundOcc`, arXiv:2303.09551；`OpenOccupancy`, arXiv:2303.03991；`OccWorld`, arXiv:2311.16038 | 发展中 |

## Autonomous Driving

| 方向 | 代表工作 | 成熟度 |
| --- | --- | --- |
| Planning-oriented E2E | `UniAD`, arXiv:2212.10156；`VAD`, arXiv:2303.12077 | 研究成熟 |
| Behavior Cloning | `PilotNet`, arXiv:1604.07316；`ChauffeurNet`, arXiv:1812.03079；`TCP`, arXiv:2206.08129 | 成熟 baseline |
| AD VLM/VLA | `DriveLM`, arXiv:2312.14150；`DriveVLM`（慢-快双系统）, arXiv:2402.12289；`EMMA`, arXiv:2410.23262；`OpenDriveVLA`, AAAI 2026, arXiv:2503.23463；`AutoVLA`, arXiv:2506.13757 | 2024-2026 前沿 |
| AD VLA 综述 | `Vision-Language-Action Models for Autonomous Driving: Past, Present, and Future`, arXiv:2512.16760 | 2025 综述 |
| AD World Model | `GAIA-1`, arXiv:2309.17080；`GAIA-2`, arXiv:2503.20523；`Drive-WM`, arXiv:2311.17918；`Vista`, NeurIPS 2024, arXiv:2405.17398；`BEVWorld`, arXiv:2407.05679 | 前沿 |
| Benchmark / simulation | `nuPlan`, arXiv:2106.11810；`NAVSIM`, arXiv:2406.15349；`ReSim`, arXiv:2506.09981 | 发展中 |

## Robotics and VLA

| 方向 | 代表工作 | 成熟度 |
| --- | --- | --- |
| RT series | `RT-1`, arXiv:2212.06817；`RT-2`, arXiv:2307.15818；`Open X-Embodiment / RT-X`, arXiv:2310.08864；`RT-H`, arXiv:2403.01823 | 研究成熟 |
| Open VLA | `OpenVLA`, arXiv:2406.09246 | 开源 baseline |
| Action representation | `FAST`（DCT+BPE action tokenizer）, arXiv:2501.09747；`Diffusion Policy`（连续动作）, arXiv:2303.04137；`SmolVLA`, arXiv:2506.01844 | 2025 前沿 |
| Generalist robot model | `π0`（flow matching）, arXiv:2410.24164；`π0.5`, arXiv:2504.16054；`GR00T N1`（dual-system）, arXiv:2503.14734；`GR00T N1.5`, 2025；`Figure Helix`, 2025 | 2024-2025 前沿 |
| Robot world model | `Diffusion Policy`, arXiv:2303.04137；`FLARE`, arXiv:2505.15659 | 发展中 |

## World Model and Generative

| 方向 | 代表工作 | 成熟度 |
| --- | --- | --- |
| Latent World Model | `World Models`, Ha & Schmidhuber, 2018, arXiv:1803.10122；`DreamerV3`, arXiv:2301.04104；`V-JEPA 2`, arXiv:2506.09985 | 成熟研究路线 |
| Video World Model | OpenAI `Sora` technical note, 2024；Google DeepMind `Genie 2`, 2024；`Genie 3`, 2025；NVIDIA `Cosmos`, arXiv:2501.03575 | 前沿系统 |
| BEV / Occupancy World Model | `BEVWorld`, arXiv:2407.05679；`OccWorld`, arXiv:2311.16038；`GEM`, CVPR 2025, arXiv:2412.11198 | 前沿 |
| Edge-cloud World Model | Waymo World Model, 2026；NVIDIA `Cosmos`, arXiv:2501.03575 | 产业前沿 |

## Workload Characterization

本页是代表来源索引。条目选择标准是：能代表一个算法范式、能解释一个 workload 瓶颈，或属于 2025-2026 需要持续跟踪的前沿方向。
