# Reading Roadmap by Goal

上级：[Reference](README.md)
相关：[Multi-dimensional Index](multi-dimensional-index.md), [Workload Comparison Table](workload-comparison-table.md)

## 这页在回答什么问题

这页按学习目标给出阅读路径。不同目标不需要从 00 顺序读到 07；架构探索、算法补课、前沿跟踪和芯片需求整理应该走不同路径。

## 目标：建立 workload 分析方法

1. [Workload Lens](../00-overview/workload-lens.md)
2. [Workload Analysis Methodology](../06-chip-workload-analysis/workload-analysis-methodology.md)
3. [Workload Characterization Axes](../06-chip-workload-analysis/workload-characterization-axes.md)
4. [Workload Comparison Table](workload-comparison-table.md)
5. [AD and Robotics Chip Architecture Summary](../06-chip-workload-analysis/ad-robotics-chip-architecture-summary.md)

这条路径适合先建立统一口径，再回头看具体算法。

## 目标：为 deterministic NPU 做架构探索

1. [CNN Workload](../06-chip-workload-analysis/cnn-workload.md)
2. [Transformer Workload](../06-chip-workload-analysis/transformer-workload.md)
3. [BEV Workload](../06-chip-workload-analysis/bev-workload.md)
4. [Occupancy Workload](../06-chip-workload-analysis/occupancy-workload.md)
5. [Edge Inference Chip Requirements](../06-chip-workload-analysis/edge-inference-chip-requirements.md)
6. [AD and Robotics Chip Architecture Summary](../06-chip-workload-analysis/ad-robotics-chip-architecture-summary.md)

读完后应能把 Resource、Topology、Interaction、Capability 的扫描变量列出来。

## 目标：补齐自动驾驶算法路线

1. [BEV Perception](../02-vision-and-3d-perception/bev-perception.md)
2. [Modular to BEV Evolution](../03-autonomous-driving-algorithms/modular-to-bev-evolution.md)
3. [Planning-oriented E2E](../03-autonomous-driving-algorithms/planning-oriented-e2e.md)
4. [Behavior Cloning E2E](../03-autonomous-driving-algorithms/behavior-cloning-e2e.md)
5. [VLM and VLA for Autonomous Driving](../03-autonomous-driving-algorithms/vlm-vla-for-ad.md)
6. [World Model for Autonomous Driving](../03-autonomous-driving-algorithms/world-model-for-ad.md)
7. [E2E Workload](../06-chip-workload-analysis/e2e-workload.md)

## 目标：补齐机器人 VLA

1. [VLA Fundamentals](../04-robotics-and-vla/vla-fundamentals.md)
2. [Action Tokenizer](../04-robotics-and-vla/action-tokenizer.md)
3. [RT Series](../04-robotics-and-vla/rt-series.md)
4. [OpenVLA](../04-robotics-and-vla/openvla.md)
5. [GR00T](../04-robotics-and-vla/groot.md)
6. [Robot World Model](../04-robotics-and-vla/robot-world-model.md)
7. [VLA Workload](../06-chip-workload-analysis/vla-workload.md)

## 目标：理解 World Model

1. [World Model Fundamentals](../05-world-model-and-generative/world-model-fundamentals.md)
2. [World Model Is Not Video Generation](../05-world-model-and-generative/world-model-is-not-video-generation.md)
3. [Latent World Model](../05-world-model-and-generative/latent-world-model.md)
4. [Video World Model](../05-world-model-and-generative/video-world-model.md)
5. [BEV World Model](../05-world-model-and-generative/bev-world-model.md)
6. [Occupancy World Model](../05-world-model-and-generative/occupancy-world-model.md)
7. [World Model Workload](../06-chip-workload-analysis/world-model-workload.md)

## 目标：维护前沿来源

1. [Source Tracking](source-tracking.md)
2. [Representative Papers](representative-papers.md)
3. [VLA Workload](../06-chip-workload-analysis/vla-workload.md)
4. [World Model Workload](../06-chip-workload-analysis/world-model-workload.md)
5. [Cloud Inference and Simulation Chip](../06-chip-workload-analysis/cloud-inference-and-simulation-chip.md)

前沿来源的维护重点是 VLA、Action Tokenizer、World Model、E2E AD、robot foundation model。更新时要同步检查 03-06 的引用口径。

## Workload Characterization

本页是阅读路径，不代表单一 workload。它的作用是把不同目标映射到最短阅读链路，减少重复阅读。
