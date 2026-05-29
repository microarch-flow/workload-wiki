# GR00T

上级：[Robotics and VLA](README.md)
相关：[VLA Fundamentals](vla-fundamentals.md), [Robot World Model](robot-world-model.md), [VLA Workload](../06-chip-workload-analysis/vla-workload.md)

## 这页在回答什么问题

这页拆解 NVIDIA GR00T N1 的定位。GR00T N1 是面向 generalist humanoid robots 的 foundation model，重点不只是机械臂操作，而是 humanoid embodiment、双臂、移动、语言任务和仿真数据闭环的组合。

## 系统定位

GR00T N1 在 2025 年公开，目标是提供可定制的 humanoid robot foundation model。它把视觉语言理解、机器人状态和动作生成结合起来，并强调 synthetic data、simulation framework 和真实机器人数据的协同。

与 OpenVLA 相比，GR00T 更偏产业化 humanoid 平台：不仅要做桌面 manipulation，还要覆盖多 embodiment、多任务、双臂和全身控制。它的 workload 也因此更复杂：输入更多、动作维度更高、控制层级更深、仿真训练闭环更重。

## 架构含义

从 workload 角度看，GR00T 类模型可以拆成三层：

```text
vision-language reasoning
   ->
embodied state / task representation
   ->
humanoid action policy
```

第一层接近 VLM，处理图像、语言和任务上下文。第二层把视觉语言表示对齐到机器人本体状态。第三层输出可执行动作，可能包含手臂、手指、底盘、躯干等多个控制通道。

常见误解：humanoid foundation model 只是把 VLA 换到人形机器人上。实际上，人形机器人增加了动力学、平衡、双臂协调、接触和安全约束，action head 与低层控制的耦合显著更强。

## 成熟度判断

GR00T N1 属于 2025 产业研究原型。它显示了 humanoid VLA 的方向，但公开资料不能等同于大规模量产验证。更现实的判断是：仿真、数据生成、少量机器人任务和开发者生态正在成熟；开放环境下的长期自主任务仍是前沿。

## 一句话理解

GR00T 把 VLA 从机械臂操作推向 humanoid foundation model；它把 workload 从单臂视觉控制扩展到多输入、多动作通道、仿真数据闭环和更严格的实时安全约束。

## Workload Characterization

- 计算密度：VLM backbone、state fusion、multi-branch action policy 共同构成主计算；humanoid action head 比单臂任务更复杂。
- 访存模式：多相机、robot proprioception、任务 token、历史状态和动作 chunk 需要持续缓存；仿真训练侧数据量大。
- 并行性：传感器编码、动作分支和仿真场景可并行；单机器人闭环控制受时间步依赖限制。
- 数据复用：共享 embodied representation 可复用于双臂、抓取、导航和任务判断；仿真生成数据可回流训练。
- 量化敏感度：语义 backbone 可量化；平衡、接触、手指控制和安全边界相关输出需要保守处理。
- 瓶颈类型：训练侧是仿真/真实数据吞吐和模型规模；部署侧是实时控制、内存容量和多传感器 IO。
- 对硬件的核心需求：高效 VLM 推理、多流传感器输入、动作 head 低延迟、多控制通道同步、仿真训练集群吞吐。

## 参考来源

- NVIDIA, `GR00T N1: An Open Foundation Model for Generalist Humanoid Robots`, arXiv:2503.14734，https://arxiv.org/abs/2503.14734，成熟度：2025 产业研究原型，查证日期：2026-05-29。
- NVIDIA Research, `GR00T N1` project page, https://research.nvidia.com/labs/lpr/publication/gr00tn1_2025/，成熟度：官方研究资料，查证日期：2026-05-29。
- NVIDIA Newsroom, `NVIDIA Announces Isaac GR00T N1`, 2025-03-18，https://nvidianews.nvidia.com/news/nvidia-isaac-gr00t-n1-open-humanoid-robot-foundation-model-simulation-frameworks，成熟度：官方产品/生态发布，查证日期：2026-05-29。
- NVIDIA Research, `GR00T N1.5`, https://research.nvidia.com/labs/gear/gr00t-n1_5/，成熟度：2025-2026 演进方向，查证日期：2026-05-29。

## 旧版素材

- `/mnt/e/workload-wiki-old/04_机器人与VLA/GR00T模型拆解.md`
