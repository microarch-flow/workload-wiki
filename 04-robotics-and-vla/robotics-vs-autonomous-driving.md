# Robotics vs Autonomous Driving

上级：[Robotics and VLA](README.md)
相关：[Autonomous Driving Algorithms](../03-autonomous-driving-algorithms/README.md), [AD Robotics Summary](../06-chip-workload-analysis/ad-robotics-chip-architecture-summary.md)

## 这页在回答什么问题

这页回答机器人和自动驾驶为什么可以放在同一个 workload 视角下比较，以及它们的差异为什么会导致不同芯片需求。两者都是 embodied AI，但 action space、传感器、时序和安全约束并不相同。

## 共性

机器人和自动驾驶共享四个趋势：

- 从模块化 pipeline 走向更统一的端到端 policy。
- 从单帧感知走向视频、历史状态和 world model。
- 从固定类别识别走向开放语义和语言/任务条件。
- 从离线指标走向 closed-loop evaluation。

因此两者都会产生多模态 token、temporal cache、action decoder、simulation/training loop 等 workload。

## 差异

| 维度 | 自动驾驶 | 机器人 |
| --- | --- | --- |
| 动作空间 | 低维轨迹、速度、转向 | 高维关节、末端姿态、夹爪、双臂 |
| 环境 | 大范围交通场景 | 近场接触和物体操作 |
| 语言作用 | 多为解释、指令、辅助 reasoning | 直接定义任务目标 |
| 控制频率 | planning 可较低频，控制器高频 | policy 与低层控制耦合更紧 |
| 数据来源 | fleet log 规模大 | 机器人演示昂贵、embodiment 分散 |
| 安全约束 | 交通规则、碰撞、可行驶区域 | 接触力、夹持、碰撞、跌倒、设备损坏 |

## 对 workload 的影响

自动驾驶中，BEV、Occupancy、trajectory planning 和多传感器同步是核心。机器人中，VLA backbone、action tokenizer、proprioception、wrist camera、contact-rich manipulation 和高频闭环控制更突出。

自动驾驶动作输出通常是未来几秒 ego trajectory；机器人动作可能是 10-50Hz 的 action chunk，也可能进一步交给 kHz 级低层控制器。这个差异决定了机器人对 action decoder latency 和稳定性的要求更高。

## 常见误解

常见误解：自动驾驶和机器人只是场景不同，端到端模型可以直接复用。实际上，二者共享 perception/temporal/world model 组件，但 action representation 和安全闭环完全不同；尤其机器人接触动力学和跨 embodiment action space 不能靠改 prompt 解决。

## 一句话理解

自动驾驶和机器人共享 embodied AI 的多模态时序 workload，但机器人把压力更多放在高维动作、接触闭环和跨本体泛化上。

## Workload Characterization

- 计算密度：两者都包含视觉主干和时序模型；自动驾驶更重 BEV/occupancy，机器人更重 VLA/action decoder。
- 访存模式：自动驾驶多路相机和 BEV cache 显著；机器人多相机、robot state、KV cache、action chunk 更显著。
- 并行性：自动驾驶可按 camera/BEV cell/task head 并行；机器人可按 sensor/action branch 并行，但闭环控制串行约束更强。
- 数据复用：自动驾驶复用 BEV scene feature；机器人复用视觉语言表征和短时历史状态。
- 量化敏感度：两者主干都可探索量化；自动驾驶 collision margin、机器人接触动作和末端姿态更敏感。
- 瓶颈类型：自动驾驶车端常受多传感器吞吐和 BEV 构建影响；机器人端侧常受 VLM latency、动作解码和控制稳定性影响。
- 对硬件的核心需求：自动驾驶强调多传感器融合和空间表征；机器人强调低 batch VLA、动作低延迟、状态融合和安全监控。

## 参考来源

- Hu et al., `Planning-oriented Autonomous Driving`, CVPR 2023 / arXiv:2212.10156，https://arxiv.org/abs/2212.10156，成熟度：自动驾驶 E2E 代表，查证日期：2026-05-29。
- Zitkovich et al., `RT-2`, CoRL 2023 / arXiv:2307.15818，https://arxiv.org/abs/2307.15818，成熟度：机器人 VLA 代表，查证日期：2026-05-29。
- Kim et al., `OpenVLA`, arXiv:2406.09246，https://arxiv.org/abs/2406.09246，成熟度：开源机器人 VLA，查证日期：2026-05-29。
- NVIDIA, `GR00T N1`, arXiv:2503.14734，https://arxiv.org/abs/2503.14734，成熟度：humanoid VLA 前沿，查证日期：2026-05-29。
