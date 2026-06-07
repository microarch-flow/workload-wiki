# Research Questions

上级：[Overview](README.md)
相关：[Workload Lens](workload-lens.md), [Wiki Roadmap](wiki-roadmap.md), [AD and Robotics Chip Architecture Summary](../06-chip-workload-analysis/ad-robotics-chip-architecture-summary.md)

## 这页在回答什么问题

这页写清这份 wiki 要回答的核心问题，以及"读完算成功"的标准。它不是泛泛的学习目标，而是架构探索时需要被回答的具体问题——每个问题都能映射到某一章。

## 这份 wiki 的根问题

一句话：**作为 AI 加速器架构师，怎么把自动驾驶和机器人场景里的算法，翻译成 deterministic NPU 架构探索可以建模的 workload？**

注意这不是"怎么成为算法专家"。学算法是手段，看出硬件需求是目的。一切深度都校准到这个目的：算法原理讲到能看出 workload 即可，不陷进训练技巧和损失函数推导。

## 拆成可回答的子问题

围绕根问题，这份 wiki 要让人能回答下面这些具体问题。

算法计算结构层面：CNN 的不同卷积形态为什么数据复用差异巨大（[CNN Workload](../06-chip-workload-analysis/cnn-workload.md)）？Transformer 推理为什么必须拆成 prefill 和 decode 两个特征相反的阶段（[Transformer Workload](../06-chip-workload-analysis/transformer-workload.md)）？FlashAttention 的 online softmax 为什么把 attention 从访存问题拉回算力问题，KV cache 压缩（GQA/MQA/MLA）为什么从推理 trick 上升为模型结构（[Attention Variants and Efficiency](../01-foundation-model-components/attention-variants-and-efficiency.md)）？Mamba/SSM 的状态递推把长序列压力从 attention matrix 转移到了哪里，为什么它是 dense-GEMM 和 full-attention 之外的第三类算子、需要 archax 在 Capability 轴预留（[Mamba and SSM](../01-foundation-model-components/mamba-and-ssm.md)）？Diffusion 的成本为什么是单步乘以步数乘以候选数（[Diffusion Models](../01-foundation-model-components/diffusion-models.md)）？

感知与系统层面：BEV 的瓶颈为什么从 backbone 算力转移到了 view transform 的不规则访存，LSS 的 forward scatter 和 BEVFormer 的 backward gather 各自的硬件代价是什么（[BEV Workload](../06-chip-workload-analysis/bev-workload.md)）？为什么检测/分割正从"专用 head + NMS/动态实例数"收敛到"统一 scene query + set prediction"，这对去掉 CPU/NPU 同步点意味着什么（[Object Detection](../02-vision-and-3d-perception/object-detection.md)、[Instance Segmentation](../02-vision-and-3d-perception/instance-segmentation.md)）？Occupancy 为什么是容量优先的 workload，dense 和 sparse 各自的硬件代价是什么（[Occupancy Workload](../06-chip-workload-analysis/occupancy-workload.md)）？端到端自动驾驶为什么成败在 worst-case 延迟和 CPU/NPU 同步而非峰值算力（[E2E Workload](../06-chip-workload-analysis/e2e-workload.md)）？

前沿方向层面：VLA 为什么是大模型推理和高频实时控制的叠加，action 输出走自回归 token（含 FAST 频域压缩）还是 flow/diffusion chunk 对 workload 影响如何，慢-快双系统为什么要求一颗芯片并发承载两个相反工作点（[VLA Workload](../06-chip-workload-analysis/vla-workload.md)）？World Model 和视频生成的根本区别是什么，latent 预测式（上端）和生成式 video（上云）为什么是两套几乎相反的硬件需求（[World Model Workload](../06-chip-workload-analysis/world-model-workload.md)）？为什么 SSM 状态递推、diffusion 多步、world model rollout、video 流式状态可归为同一类"带状态的迭代推理"，它在 archax Interaction 轴上是哪个维度？这些 2024-2026 快速演化的方向，当前真实状态（已落地/论文/概念）到底是什么（[Source Tracking](../07-reference/source-tracking.md)）？

硬件映射层面：端侧和云端的 workload 约束为什么几乎相反，分别该怎么设计（[Edge Inference Chip Requirements](../06-chip-workload-analysis/edge-inference-chip-requirements.md)、[Cloud Inference and Simulation Chip](../06-chip-workload-analysis/cloud-inference-and-simulation-chip.md)）？这些 workload 怎么落到 RAM/DMA/NOC/CIM/PCIE 和 archax 的 Resource/Topology/Interaction/Capability（[AD and Robotics Chip Architecture Summary](../06-chip-workload-analysis/ad-robotics-chip-architecture-summary.md)）？

## 成功标准

读完这份 wiki 后，应该能对任意一个目标算法，不查资料就说出它的 workload 画像：计算密度、访存模式、并行性、数据复用、量化敏感度、第一/第二瓶颈、对硬件的核心需求；并且能说出它该映射到 NPU 架构的哪些扫描变量上（SRAM 容量、DMA 能力、NOC QoS、cache residency、低比特模式、candidate 并发度、CPU/NPU 同步路径等），而不是只报一个 TOPS 数字。

## 一句话理解

这份 wiki 成功与否，看的是读完之后能不能把一个算法直接说成一组芯片约束和一组 archax 扫描变量。
