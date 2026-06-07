# Multi-dimensional Index

上级：[Reference](README.md)
相关：[Workload Comparison Table](workload-comparison-table.md), [Workload Analysis Methodology](../06-chip-workload-analysis/workload-analysis-methodology.md)

## 这页在回答什么问题

这页提供多维索引。读者可以按算法类别、应用场景、workload 瓶颈、硬件资源或 archax 建模对象进入 wiki，而不必按章节线性阅读。

## 按算法类别

| 类别 | 入口 |
| --- | --- |
| CNN / backbone | [CNN Backbone](../01-foundation-model-components/cnn-backbone.md), [CNN Workload](../06-chip-workload-analysis/cnn-workload.md) |
| Transformer / Attention | [Attention and Transformer](../01-foundation-model-components/attention-and-transformer.md), [Transformer Workload](../06-chip-workload-analysis/transformer-workload.md) |
| Diffusion | [Diffusion Models](../01-foundation-model-components/diffusion-models.md), [Diffusion for World Model](../05-world-model-and-generative/diffusion-for-world-model.md) |
| Mamba / SSM | [Mamba and SSM](../01-foundation-model-components/mamba-and-ssm.md) |
| BEV / Occupancy | [BEV Perception](../02-vision-and-3d-perception/bev-perception.md), [Occupancy Prediction](../02-vision-and-3d-perception/occupancy-prediction.md) |
| VLA | [VLA Fundamentals](../04-robotics-and-vla/vla-fundamentals.md), [VLA Workload](../06-chip-workload-analysis/vla-workload.md) |
| World Model | [World Model Fundamentals](../05-world-model-and-generative/world-model-fundamentals.md), [World Model Workload](../06-chip-workload-analysis/world-model-workload.md) |

## 按算法概念

这一节把 01-05 反复出现的关键机制做成可检索入口，方便从"概念"反查到展开页面和对应 workload 性格。

| 概念 | 含义 / workload 性格 | 入口 |
| --- | --- | --- |
| selective scan / SSM / 状态递推 | 线性时间序列算子，stateful decode，省 KV cache | [Mamba and SSM](../01-foundation-model-components/mamba-and-ssm.md) |
| hybrid（attention-SSM 混层） | 多数层 SSM 扛吞吐、少数层 attention 做精确检索（Jamba） | [Mamba and SSM](../01-foundation-model-components/mamba-and-ssm.md), [Attention and Transformer](../01-foundation-model-components/attention-and-transformer.md) |
| GQA / MQA / MLA | KV cache 结构性压缩，缓解 decode 的 bandwidth-bound | [Attention Variants and Efficiency](../01-foundation-model-components/attention-variants-and-efficiency.md), [Transformer Workload](../06-chip-workload-analysis/transformer-workload.md) |
| token 治理（merge / prune / 变长 packing） | 视觉 token 数作为一级可调旋钮（NaViT），动态 shape | [Vision Transformer Backbone](../01-foundation-model-components/vision-transformer-backbone.md) |
| set prediction（NMS-free） | DETR 类 query 直接出集合，去掉后处理 | [Object Detection](../02-vision-and-3d-perception/object-detection.md) |
| mask classification | Mask2Former 类统一分割范式 | [Semantic Segmentation](../02-vision-and-3d-perception/semantic-segmentation.md), [Instance Segmentation](../02-vision-and-3d-perception/instance-segmentation.md) |
| view transform（LSS scatter vs BEVFormer gather） | BEV 投影的两种访存模式，决定 scatter-gather 取舍 | [BEV Perception](../02-vision-and-3d-perception/bev-perception.md), [BEV Workload](../06-chip-workload-analysis/bev-workload.md) |
| covariate shift | behavior cloning 的分布漂移，闭环数据/扰动对治 | [Behavior Cloning E2E](../03-autonomous-driving-algorithms/behavior-cloning-e2e.md) |
| dual-system 慢-快（System 2 / System 1） | VLM 慢思考 + action expert 快控制（DriveVLM、GR00T N1） | [VLM and VLA for Autonomous Driving](../03-autonomous-driving-algorithms/vlm-vla-for-ad.md), [VLA Fundamentals](../04-robotics-and-vla/vla-fundamentals.md) |
| action tokenizer（FAST 的 DCT+BPE） | 频域压缩动作 token，缩短 decode 序列 | [Action Tokenizer](../04-robotics-and-vla/action-tokenizer.md) |
| flow matching 连续动作 | 不走 token，一次出动作 chunk（π0、GR00T N1） | [Action Tokenizer](../04-robotics-and-vla/action-tokenizer.md), [VLA Fundamentals](../04-robotics-and-vla/vla-fundamentals.md) |
| action chunking | 一次预测 H 步动作，摊薄推理频率 | [VLA Fundamentals](../04-robotics-and-vla/vla-fundamentals.md), [VLA Workload](../06-chip-workload-analysis/vla-workload.md) |
| latent vs generative world model | latent 预测省像素解码 vs 像素生成（V-JEPA 2 vs Cosmos/GAIA） | [World Model Is Not Video Generation](../05-world-model-and-generative/world-model-is-not-video-generation.md), [Latent World Model](../05-world-model-and-generative/latent-world-model.md) |

## 按应用场景

| 场景 | 入口 |
| --- | --- |
| 视觉感知 | [Object Detection](../02-vision-and-3d-perception/object-detection.md), [Semantic Segmentation](../02-vision-and-3d-perception/semantic-segmentation.md), [Instance Segmentation](../02-vision-and-3d-perception/instance-segmentation.md) |
| 3D 感知 | [LiDAR Point Cloud Processing](../02-vision-and-3d-perception/lidar-point-cloud-processing.md), [Multi-sensor Fusion](../02-vision-and-3d-perception/multi-sensor-fusion.md), [Occupancy Prediction](../02-vision-and-3d-perception/occupancy-prediction.md) |
| 自动驾驶 E2E | [Planning-oriented E2E](../03-autonomous-driving-algorithms/planning-oriented-e2e.md), [Behavior Cloning E2E](../03-autonomous-driving-algorithms/behavior-cloning-e2e.md), [E2E Workload](../06-chip-workload-analysis/e2e-workload.md) |
| 机器人 VLA | [RT Series](../04-robotics-and-vla/rt-series.md), [OpenVLA](../04-robotics-and-vla/openvla.md), [GR00T](../04-robotics-and-vla/groot.md) |
| 仿真与数据闭环 | [Data Closed Loop and Simulation](../03-autonomous-driving-algorithms/data-closed-loop-and-simulation.md), [Edge-cloud Collaborative World Model](../05-world-model-and-generative/edge-cloud-collaborative-world-model.md) |

## 按第一瓶颈

| 第一瓶颈 | 典型 workload | 入口 |
| --- | --- | --- |
| compute-bound | standard CNN、Transformer prefill、large FFN | [CNN Workload](../06-chip-workload-analysis/cnn-workload.md), [Transformer Workload](../06-chip-workload-analysis/transformer-workload.md) |
| memory-bandwidth-bound | Transformer decode、depthwise conv、KV cache | [Transformer Workload](../06-chip-workload-analysis/transformer-workload.md) |
| irregular-access-bound | BEV view transform、sparse point/voxel | [BEV Workload](../06-chip-workload-analysis/bev-workload.md), [Occupancy Workload](../06-chip-workload-analysis/occupancy-workload.md) |
| memory-capacity-bound | 3D occupancy、long context、multi-frame BEV | [Occupancy Workload](../06-chip-workload-analysis/occupancy-workload.md), [World Model Workload](../06-chip-workload-analysis/world-model-workload.md) |
| latency-bound | robot action decode、E2E planning、safety shell | [VLA Workload](../06-chip-workload-analysis/vla-workload.md), [E2E Workload](../06-chip-workload-analysis/e2e-workload.md) |
| synchronization-bound | CPU/NPU safety、multi-stage E2E pipeline | [Edge Inference Chip Requirements](../06-chip-workload-analysis/edge-inference-chip-requirements.md) |

## 按硬件资源

| 硬件资源 | 重点页面 |
| --- | --- |
| RAM / SRAM / DRAM | [Workload Characterization Axes](../06-chip-workload-analysis/workload-characterization-axes.md), [Occupancy Workload](../06-chip-workload-analysis/occupancy-workload.md) |
| DMA | [BEV Workload](../06-chip-workload-analysis/bev-workload.md), [Edge Inference Chip Requirements](../06-chip-workload-analysis/edge-inference-chip-requirements.md) |
| NOC | [E2E Workload](../06-chip-workload-analysis/e2e-workload.md), [AD and Robotics Chip Architecture Summary](../06-chip-workload-analysis/ad-robotics-chip-architecture-summary.md) |
| CIM | [CNN Workload](../06-chip-workload-analysis/cnn-workload.md), [Transformer Workload](../06-chip-workload-analysis/transformer-workload.md) |
| PCIE / host boundary | [Cloud Inference and Simulation Chip](../06-chip-workload-analysis/cloud-inference-and-simulation-chip.md), [Edge Inference Chip Requirements](../06-chip-workload-analysis/edge-inference-chip-requirements.md) |

## 按 archax 建模对象

| archax 对象 | 重点问题 | 入口 |
| --- | --- | --- |
| Resource | compute、SRAM、DRAM/HBM、DMA、NoC、CIM、host interface 的容量和吞吐 | [Workload Analysis Methodology](../06-chip-workload-analysis/workload-analysis-methodology.md), [Edge Inference Chip Requirements](../06-chip-workload-analysis/edge-inference-chip-requirements.md) |
| Topology | NPU tile、memory hierarchy、sensor ingress、controller、cloud fabric 如何连接 | [AD and Robotics Chip Architecture Summary](../06-chip-workload-analysis/ad-robotics-chip-architecture-summary.md), [Cloud Inference and Simulation Chip](../06-chip-workload-analysis/cloud-inference-and-simulation-chip.md) |
| Interaction | tensor transfer、cache update、decode loop、rollout loop、CPU/NPU sync | [Transformer Workload](../06-chip-workload-analysis/transformer-workload.md), [World Model Workload](../06-chip-workload-analysis/world-model-workload.md), [E2E Workload](../06-chip-workload-analysis/e2e-workload.md) |
| Capability | Conv/GEMM/Attention、scatter-gather、sparse metadata、quantization、stateful decode | [CNN Workload](../06-chip-workload-analysis/cnn-workload.md), [BEV Workload](../06-chip-workload-analysis/bev-workload.md), [Occupancy Workload](../06-chip-workload-analysis/occupancy-workload.md), [VLA Workload](../06-chip-workload-analysis/vla-workload.md) |

## Workload Characterization

本页是索引，不是单一 workload。它按瓶颈类型和硬件资源把具体 workload 页面串起来，方便从架构问题反查算法来源。
