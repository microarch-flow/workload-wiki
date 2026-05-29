# Robotics and VLA

上级：[Workload Wiki](../README.md)
相关：[VLA Workload](../06-chip-workload-analysis/vla-workload.md), [World Model Workload](../06-chip-workload-analysis/world-model-workload.md)

## 这章在回答什么问题

这一章回答机器人为什么从 task-specific pipeline 走向 Vision-Language-Action model，以及这种路线对 workload 的影响。机器人 VLA 的关键不是“加了语言输入”，而是把视觉观察、任务语义、历史状态和动作输出放进统一模型接口。

## 主线

```text
classic robotics pipeline
  -> imitation learning / behavior cloning
  -> RT series and multi-robot datasets
  -> OpenVLA / GR00T / pi0-style generalist policy
  -> action tokenizer / flow or diffusion action head
  -> robot world model
```

机器人和自动驾驶都在走向多模态、时序、端侧闭环推理。但机器人动作空间更高维、更连续、更依赖接触反馈；因此它对 action representation、低延迟控制和跨 embodiment 泛化更敏感。

## 页面列表

- [VLA Fundamentals](vla-fundamentals.md)
- [Action Tokenizer](action-tokenizer.md)
- [RT Series](rt-series.md)
- [OpenVLA](openvla.md)
- [GR00T](groot.md)
- [Robot World Model](robot-world-model.md)
- [Robotics vs Autonomous Driving](robotics-vs-autonomous-driving.md)

## 到 06 的映射

| 页面 | 主要 workload | 06 对应入口 |
| --- | --- | --- |
| VLA Fundamentals | visual token、language token、robot state、action head | [VLA Workload](../06-chip-workload-analysis/vla-workload.md) |
| Action Tokenizer | action discretization、autoregressive decode、continuous control recovery | [VLA Workload](../06-chip-workload-analysis/vla-workload.md) |
| RT / OpenVLA / GR00T | VLM backbone、robot action decoder、multi-embodiment policy | [VLA Workload](../06-chip-workload-analysis/vla-workload.md) |
| Robot World Model | future state rollout、video/latent dynamics、policy evaluation | [World Model Workload](../06-chip-workload-analysis/world-model-workload.md) |
| Robotics vs AD | shared embodied workload vs embodiment-specific workload | [AD Robotics Summary](../06-chip-workload-analysis/ad-robotics-chip-architecture-summary.md) |

## 阅读方式

先读 VLA 基础和 Action Tokenizer，建立“动作如何进入大模型”的概念；再读 RT、OpenVLA、GR00T，理解代表系统的架构演进；最后读 Robot World Model 和机器人/自动驾驶对比，把机器人 workload 放回统一芯片需求视角。

## 旧版素材

- `/mnt/e/workload-wiki-old/04_机器人与VLA/机器人与VLA总览.md`
