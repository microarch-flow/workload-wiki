# Data Closed Loop and Simulation

上级：[Autonomous Driving Algorithms](README.md)
相关：[Cloud Inference and Simulation Chip](../06-chip-workload-analysis/cloud-inference-and-simulation-chip.md), [World Model for Autonomous Driving](world-model-for-ad.md)

## 这页在回答什么问题

这页回答自动驾驶算法为什么离不开数据闭环与仿真。随着 E2E、VLM/VLA 和 World Model 进入系统，瓶颈不只是模型结构，而是如何持续发现长尾、重放失败、生成数据、训练模型并在闭环中评估。

## 数据闭环

典型数据闭环包括：

```text
fleet logs
   ->
scenario mining / auto labeling / failure triage
   ->
training data curation
   ->
model training and validation
   ->
simulation / closed-loop evaluation
   ->
deployment and monitoring
```

闭环的核心是把真实世界中的失败、接管、near miss、规则冲突和 rare event 转换成可训练、可回放、可评估的数据资产。

## 仿真的层次

| 层次 | 内容 | 适合用途 |
| --- | --- | --- |
| log replay | 重放真实传感器或中间表示 | 回归测试、离线评估 |
| non-reactive simulation | ego 改变，环境基本不响应 | 快速 benchmark、planning 对比 |
| reactive simulation | 其他 agent 响应 ego | 闭环评估、交互场景 |
| generative simulation | world model / diffusion 生成未来 | 长尾扩增、what-if 测试 |

CARLA、nuPlan、NAVSIM 等工具分别覆盖了不同层级。World Model 的引入让仿真从规则/资产驱动进一步扩展到数据驱动生成。

## 对 E2E 的意义

E2E 模型需要闭环评估，因为 open-loop 轨迹误差无法充分反映真实驾驶安全。一个轨迹在 log 中看起来接近专家，不代表当它导致旁车响应变化时仍安全。

因此，E2E 时代的数据系统要能支持：

- 多版本模型大规模回放；
- 自动挖掘误差和长尾；
- 生成 counterfactual scenario；
- 统计安全、舒适、进度、规则等指标；
- 把失败样本回流到训练集。

## 成熟度判断

- 成熟：log replay、自动标注、离线训练数据闭环。
- 发展中：nuPlan/NAVSIM 类 closed-loop 或 proxy closed-loop benchmark。
- 前沿：reactive/generative simulation 与 World Model 结合，尤其是可验证的 long-tail counterfactual generation。

## 一句话理解

数据闭环和仿真把自动驾驶从“训练一个模型”变成“持续生产、筛选、验证驾驶经验”的系统工程；它也是 E2E 和 World Model 能否落地的基础设施。

## Workload Characterization

- 计算密度：云端回放、自动标注、仿真渲染、生成式扩增和多版本评测消耗巨大；车端主要是日志采集和触发式上传。
- 访存模式：多传感器日志、标注、中间 feature、仿真状态和模型输出需要高吞吐读写；数据湖 IO 常成为瓶颈。
- 并行性：场景级、模型版本级、candidate policy 级高度并行；单个 reactive simulation 内部受时间步依赖限制。
- 数据复用：同一 log 可复用于感知训练、planner 评测、VLM 标注、World Model 学习和 regression suite。
- 量化敏感度：云端标注和仿真可混合精度；安全评估和 metric 计算需要保持数值一致性。
- 瓶颈类型：主要是 cloud throughput、storage bandwidth、调度效率和数据治理；生成式仿真还会受 GPU/加速器显存制约。
- 对硬件的核心需求：高吞吐数据加载、批量推理、视频/点云解码、仿真渲染、生成模型训练、跨模型版本并发评测。

## 参考来源

- Dosovitskiy et al., `CARLA: An Open Urban Driving Simulator`, CoRL 2017 / arXiv:1711.03938，成熟度：经典开源仿真平台，查证日期：2026-05-29。
- Caesar et al., `nuPlan: A closed-loop ML-based planning benchmark for autonomous vehicles`, CVPR ADP3 2022 / arXiv:2106.11810，成熟度：常用规划闭环基准，查证日期：2026-05-29。
- Dauner et al., `NAVSIM: Data-Driven Non-Reactive Autonomous Vehicle Simulation and Benchmarking`, NeurIPS 2024 Datasets and Benchmarks / arXiv:2406.15349，成熟度：新兴 benchmark，查证日期：2026-05-29。
- `ReSim: Reliable World Simulation for Autonomous Driving`, arXiv:2506.09981，https://doi.org/10.48550/arXiv.2506.09981，成熟度：2025 前沿研究，查证日期：2026-05-29。
