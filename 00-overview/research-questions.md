# Research Questions

上级：[Overview](README.md)
相关：[Workload Lens](workload-lens.md), [Wiki Roadmap](wiki-roadmap.md), [AD and Robotics Chip Architecture Summary](../06-chip-workload-analysis/ad-robotics-chip-architecture-summary.md)

## 这页在回答什么问题

这页说明 wiki 希望帮助读者回答的算法问题，以及可以继续探索的芯片工作负载问题。具体阅读顺序见 [Wiki Roadmap](wiki-roadmap.md)。

## 这份 wiki 的根问题

**一个算法要解决什么问题？它如何表示输入、执行计算并产生输出？它在什么条件下适用，又有哪些局限？**

在此基础上，还可以追问：当输入规模、软件实现和目标硬件确定后，这个算法会形成什么工作负载？哪些资源需求和性能判断需要进一步测量或建模？

## 拆成可回答的子问题

**基础组件与模型。** Attention 如何组织信息交互？[Attention Variants and Efficiency](../01-foundation-model-components/attention-variants-and-efficiency.md) 中的方法改变了什么？[Mamba and SSM](../01-foundation-model-components/mamba-and-ssm.md) 如何使用状态建模序列？[Diffusion Models](../01-foundation-model-components/diffusion-models.md) 如何通过多步过程生成样本？

**感知任务与表示。** [Object Detection](../02-vision-and-3d-perception/object-detection.md) 和 [Instance Segmentation](../02-vision-and-3d-perception/instance-segmentation.md) 的输出和评估目标有什么不同？BEV 和 occupancy 如何表示场景？不同表示保留或舍弃了哪些信息？

**应用与评估。** 自动驾驶和机器人如何把感知、预测与动作关联起来？不同训练方法需要什么数据？离线指标和闭环评估分别能说明什么，不能说明什么？阅读具体方案时，应区分模型设计、实验条件和部署情况。

**工作负载延伸。** 给定模型、输入规模和执行方式后，计算、数据访问和依赖关系如何变化？可以从 [CNN Workload](../06-chip-workload-analysis/cnn-workload.md)、[Transformer Workload](../06-chip-workload-analysis/transformer-workload.md)、[BEV Workload](../06-chip-workload-analysis/bev-workload.md) 和 [Occupancy Workload](../06-chip-workload-analysis/occupancy-workload.md) 入手，再按需阅读 [E2E Workload](../06-chip-workload-analysis/e2e-workload.md)、[VLA Workload](../06-chip-workload-analysis/vla-workload.md) 或 [World Model Workload](../06-chip-workload-analysis/world-model-workload.md)。

**硬件映射与来源。** 端侧和云侧的约束可能不同；应如何为具体场景设定延迟、吞吐、容量和功耗目标？可参考 [Edge Inference Chip Requirements](../06-chip-workload-analysis/edge-inference-chip-requirements.md)、[Cloud Inference and Simulation Chip](../06-chip-workload-analysis/cloud-inference-and-simulation-chip.md) 和 [AD and Robotics Chip Architecture Summary](../06-chip-workload-analysis/ad-robotics-chip-architecture-summary.md)。前沿工作的来源和成熟度记录见 [Source Tracking](../07-reference/source-tracking.md)。

## 成功标准

对于选定的算法，读者能够说明问题、输入和输出、关键步骤、主要假设以及适用边界，并能找到代表工作和评估依据。没有充分信息时，应能指出需要查证的部分。

如果继续做工作负载分析，读者应能列出输入规模、实现和目标硬件等前提，区分静态估算、架构模型和实测结果；不能仅凭算法名称认定性能瓶颈。

## 一句话理解

先理解算法如何解决问题，再按需要研究它在具体执行条件下形成的芯片工作负载。
