# World Model and Generative Intelligence

上级：[Workload Wiki](../README.md)
相关：[World Model Workload](../06-chip-workload-analysis/world-model-workload.md), [Diffusion Models](../01-foundation-model-components/diffusion-models.md)

## 这章在回答什么问题

这一章回答 World Model 与生成式智能的关系。这里的 World Model 不是“能生成视频的模型”的同义词，而是能预测环境状态、评估 action 后果、支撑 planning 或 simulation 的模型族。

## 主线

```text
world state representation
  -> latent dynamics / video dynamics / spatial BEV or occupancy dynamics
  -> action-conditioned rollout
  -> planning, simulation, data generation, policy training
  -> edge-cloud collaboration
```

05 章比 03/04 更抽象：它把自动驾驶和机器人中的 World Model 放进统一框架。重点是 representation 选择如何改变 workload：video token 重渲染压力大，latent dynamics 更适合 planning，BEV/occupancy 更适合自动驾驶几何验证，edge-cloud 协同决定哪些生成式计算能放在端侧。

## 页面列表

- [World Model Fundamentals](world-model-fundamentals.md)
- [World Model Is Not Video Generation](world-model-is-not-video-generation.md)
- [Video World Model](video-world-model.md)
- [Latent World Model](latent-world-model.md)
- [BEV World Model](bev-world-model.md)
- [Occupancy World Model](occupancy-world-model.md)
- [Diffusion for World Model](diffusion-for-world-model.md)
- [Edge-cloud Collaborative World Model](edge-cloud-collaborative-world-model.md)

## 到 06 的映射

| 页面 | 主要 workload | 06 对应入口 |
| --- | --- | --- |
| Fundamentals / Not Video Generation | state representation、action-conditioned rollout | [World Model Workload](../06-chip-workload-analysis/world-model-workload.md) |
| Video World Model | video tokens、diffusion/transformer generation、long horizon | [World Model Workload](../06-chip-workload-analysis/world-model-workload.md) |
| Latent World Model | latent dynamics、policy rollout、belief state | [World Model Workload](../06-chip-workload-analysis/world-model-workload.md) |
| BEV / Occupancy World Model | spatial grid、future occupancy、planning cost | [BEV Workload](../06-chip-workload-analysis/bev-workload.md), [Occupancy Workload](../06-chip-workload-analysis/occupancy-workload.md) |
| Edge-cloud Collaborative | cloud generation、edge inference、data loop | [Cloud Inference and Simulation Chip](../06-chip-workload-analysis/cloud-inference-and-simulation-chip.md) |
