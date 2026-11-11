# Representative Works Index

上级：[Overview](README.md)
相关：[Representative Papers](../07-reference/representative-papers.md), [Source Tracking](../07-reference/source-tracking.md), [Unified Relationship Map](unified-relationship-map.md)

## 这页在回答什么问题

这页按算法主题列出代表工作，帮助读者从问题和方法进入专题。它不是完整文献表，也不表示列出的方案都已部署。论文信息与来源记录见 [Representative Papers](../07-reference/representative-papers.md) 和 [Source Tracking](../07-reference/source-tracking.md)；评价具体结论时应回到原始论文及其实验条件。

<a id="基础组件定义了基本-workload-形态"></a>

## 基础组件与模型

Transformer 用 attention 处理序列中的信息交互；GQA、MQA 和 MLA 是与多头 attention 或缓存结构相关的不同设计。[FlashAttention](https://arxiv.org/abs/2205.14135) 展示了如何通过分块降低 attention 计算中的部分存储器读写。

CNN 的代表工作包括 ResNet、MobileNet 和 EfficientNet。Mamba、Mamba-2 和 Jamba 提供状态空间模型或混合结构的阅读入口。DDPM 和 Latent Diffusion 则可用于理解扩散生成。各方法的定义和条件见 [Foundation Model Components](../01-foundation-model-components/README.md)。

<a id="感知与-bevoccupancy定义了不规则访存与容量压力"></a>

## 感知与场景表示

DETR 和 Mask2Former 可用于理解以 query 为中心的检测或分割方法。Lift-Splat-Shoot、BEVFormer、SparseBEV 和 BEVFusion 是理解不同 BEV 构造或融合方法的入口。SurroundOcc、Occ3D 和 OccWorld 涉及占用表示或相关预测任务。

这些工作解决的问题和输入并不完全相同。阅读时可以对照 [BEV Perception](../02-vision-and-3d-perception/bev-perception.md) 与 [Occupancy Prediction](../02-vision-and-3d-perception/occupancy-prediction.md)，分别检查表示、监督信号和输出。

<a id="自动驾驶-e2e定义了系统级延迟与同步-workload"></a>

## 自动驾驶

[UniAD](https://arxiv.org/abs/2212.10156) 是以规划目标组织多个驾驶任务的代表工作。SparseDrive、DiffusionDrive、EMMA 和 OpenDriveVLA 可以作为了解不同表示、轨迹生成或多模态路线的入口。

这些名称不表示所有方案属于同一演进序列。具体背景见 [Autonomous Driving Algorithms](../03-autonomous-driving-algorithms/README.md)，效果比较应核对数据集、指标及实验设置。

<a id="机器人-vla定义了大模型实时控制叠加的-workload"></a>

## 机器人与 VLA

RT-2 和 OpenVLA 是理解视觉、语言与机器人动作关联方式的入口；FAST 涉及动作 token 编码。π0、π0.5、Figure Helix 和 GR00T N1 展示了其他动作生成或系统组织思路。它们的输入、动作接口、训练数据和控制方式需要分别考察。

专题入口见 [Robotics and VLA](../04-robotics-and-vla/README.md)。

<a id="world-model定义了-rollout-乘法成本"></a>

## World Model

DreamerV3 可用于了解 latent world model；GAIA-1、GAIA-2 和 Cosmos 提供生成式模型相关的阅读入口。[V-JEPA 2](https://arxiv.org/abs/2506.09985) 研究表征预测，并讨论动作条件变体的规划应用。

这些工作使用的表示、训练目标和评估任务不同，不能仅凭模型类别推断部署位置或性能。详见 [World Model and Generative](../05-world-model-and-generative/README.md)。

## 维护说明

新增工作时，说明它解决的问题、方法特点和原始来源；性能或成熟度判断应注明条件。前沿方向的来源与查证记录见 [Source Tracking](../07-reference/source-tracking.md)。如果分析芯片影响，另行说明输入规模、执行实现和目标硬件。

## 一句话理解

这页按算法问题组织代表工作，并通过专题和来源索引帮助读者继续查证。
