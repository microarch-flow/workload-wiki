# Robotics vs Autonomous Driving

上级：[Robotics and VLA](README.md)
相关：[VLA Fundamentals](vla-fundamentals.md), [VLM and VLA for Autonomous Driving](../03-autonomous-driving-algorithms/vlm-vla-for-ad.md), [Autonomous Driving Algorithms](../03-autonomous-driving-algorithms/README.md), [AD/Robotics Chip Architecture Summary](../06-chip-workload-analysis/ad-robotics-chip-architecture-summary.md)

## 这页在回答什么问题

这页回答机器人和自动驾驶为什么能放进同一个 workload 视角比较，以及它们的差异具体怎么落到不同芯片需求上。两者都是 embodied AI、都在走向多模态时序端到端 policy，所以共享一大半组件；但 action space、控制频率、本体状态、安全约束的差异，会把芯片需求推向不同方向。这页是对比综述，单侧细节见各自章节。

## 为什么能放一起比：直觉与类比

把两者放一起比的直觉是**它们是同一种"具身智能体"的两个体型**——都要"看懂世界、想清楚、动起来"，区别只在身体长什么样、动得多快、摔了多严重。对应到机制：两者的算法骨架高度同构——多模态感知出 token、时序模型压历史、一个 policy/decoder 出 action，都在从模块化 pipeline 走向端到端、从单帧走向带 world model 的时序、从闭集类别走向开放语义、从离线指标走向 closed-loop 评测。所以两者都会产生同一批 workload 原语：多模态 token、temporal cache、action decoder、simulation/training loop。这是它们能共享一颗芯片大部分通路的基础。

但"同一种智能体的两个体型"这个类比的关键，恰恰在体型差异上——而差异集中在身体那一端（action 和安全），不在脑子那一端（感知和语义）。对应到机制：感知和语义部分（vision encoder、temporal model、VLM backbone）两者可以高度复用，真正分叉的是 action representation 和安全闭环。一辆车的"动作"是低维、平滑、可预测的（未来几秒的 ego trajectory，几个自由度）；一个机械臂/人形的"动作"是高维、非线性、接触主导的（几十个关节、夹爪、接触力）。一辆车想错了有几百毫秒到几秒的余量去纠（车速有限、轨迹连续），一个机械臂的接触动作错了往往是瞬时的、不可逆的（撞坏物体、夹爆、人形跌倒）。这两点差异——动作维度和容错时间窗——是后面所有芯片需求分叉的总源头。

## 共性：可共享的那一半

机器人和自动驾驶共享四个演进趋势，因此共享一批 workload。两者都在从模块化 pipeline 走向更统一的端到端 policy；从单帧感知走向视频、历史状态和 world model；从固定类别识别走向开放语义和语言/任务条件；从离线指标走向 closed-loop evaluation。落到计算上，这意味着两者都需要：多模态 token 的编码与拼接、temporal/KV cache、action/trajectory decoder、以及训练侧的 simulation loop 和 world model（自动驾驶的 BEV/occupancy world model 与机器人的 latent world model 在"action-conditioned 未来预测"上同构，见 [Robot World Model](robot-world-model.md)）。VLM/VLA 在两侧也同源——慢-快双系统在自动驾驶（DriveVLM 类）和机器人（Helix、GR00T）独立收敛，是这种共性的最强证据。

## 差异：决定芯片需求分叉的那一半

| 维度 | 自动驾驶 | 机器人 |
| --- | --- | --- |
| 动作空间 | 低维（未来几秒 ego trajectory、速度、转向，几个自由度） | 高维（数十关节、末端姿态、夹爪、双臂，人形可达 35+ DoF） |
| 控制频率 | planning 可较低频（~10 Hz），底层控制器另在高频 | policy 与底层控制耦合更紧，快系统可达 120-200 Hz |
| 本体状态 | ego 运动状态相对简单（位姿、速度） | proprioception 复杂（关节角、力/扭矩、接触、平衡），强耦合进 action |
| 环境 | 大范围交通场景、远距离感知 | 近场接触、可操作物体、手眼协调 |
| 语言作用 | 多为解释、长尾推理、辅助 | 常直接定义任务目标 |
| 数据来源 | fleet log 海量、相对同质 | 机器人演示昂贵、embodiment 分散，需仿真/world model 补数据 |
| 安全约束 | 交通规则、碰撞、可行驶区域，容错窗口较长 | 接触力、夹持、碰撞、跌倒、设备损坏，容错窗口短到瞬时 |
| 典型 batch | 车端 batch=1，但多传感器路数多 | 端侧 batch=1，传感器路数少但 proprioception 维度高 |

这些差异怎么落到 workload。自动驾驶的计算重心在**空间表征构建**——多路相机（常 6-12 路）+ LiDAR 同步、BEV/occupancy 构建、远距离几何，访存上多传感器吞吐和 BEV cache 显著（见 03/06 对应篇）。机器人的计算重心在**高频闭环的 action 生成 + 本体状态融合**——VLA backbone、action tokenizer、wrist camera、contact-rich manipulation，访存上 KV cache + action chunk + proprioception 融合更突出。

控制频率的差异尤其硬。自动驾驶动作输出通常是未来几秒的 ego trajectory，planning 在 ~10 Hz 出一条轨迹就够，真正的高频留给传统底层控制器；机器人的 policy 动作可能是 10-50 Hz 的 action chunk，甚至双系统的快系统跑到 120-200 Hz（GR00T System 1 ~120Hz、Helix System 1 ~200Hz），再交给 kHz 级底层控制器。这决定了机器人对 action decoder 的延迟和稳定性要求显著更高——自动驾驶的大模型可以慢悠悠在双系统的慢侧出意图，机器人的快系统却要把高维动作以一两百 Hz 稳定吐出来。

## 一句话理解

自动驾驶和机器人共享 embodied AI 的多模态时序 workload（感知、temporal cache、VLA、world model 可复用大半），但机器人把压力压在高维动作、高频闭环（快系统 120-200 Hz）、复杂本体状态融合和瞬时容错的安全约束上，而自动驾驶把压力压在多传感器空间表征构建上——分叉发生在"身体"那一端，不在"脑子"那一端。

## 演进与未来方向（判断）

以下为判断，锚定 2025-2026 联网核实的真实工作，查证日期：2026-06-07。

最值得注意的趋势是**两个领域的算法骨架在加速趋同，但 action 和安全这两端在加速分化**。趋同的证据很硬：慢-快双系统在两侧独立成为主流（自动驾驶 DriveVLM/AutoVLA 类，机器人 Helix/GR00T），VLA 范式、flow/diffusion action head、world model 合成数据在两侧都在用；甚至有工作（如 NVIDIA 这类同时做两侧的玩家）试图用统一的具身基础模型同时覆盖车和机器人。分化的证据也很硬：机器人往更高频、更高 DoF、更接触密集走（人形 35+ DoF、200 Hz），自动驾驶往更远距离、更多传感器、更长时空 BEV 走。

我的判断有三。其一，**感知/语义/world model 这半边会越来越可共享，甚至共用 backbone 和工具链；action/安全这半边会顽固地按领域专用化**——因为接触动力学、跨 embodiment action space、瞬时安全这些东西改 prompt 解决不了，必须领域专门训练和验证。其二，**双系统会是两侧共同的部署形态**，但分频点不同：自动驾驶慢系统 ~10 Hz、快 planner 高频；机器人慢系统 ~10 Hz、快 policy 100-200 Hz——机器人的快慢频率差更大，并发隔离更难。其三，**"端到端模型可直接跨领域复用"是个会被反复证伪的诱惑**——共享的是组件不是系统，把车的 trajectory head 换成机械臂的 action head 远不止换个输出维度。

对架构师，这个对比的实际价值是它告诉你**一颗想同时服务两类具身负载的芯片，该把通用性放在哪、专用性放在哪**：感知/VLM/temporal/world-model 通路可以设计成两领域共享（这是统一 SoC 的机会），而 action 侧（自动驾驶的低维高频 trajectory vs 机器人的高维超高频 contact-rich action）和安全监控通路必须按目标领域专门取舍。这正是 06 [AD/Robotics Chip Architecture Summary](../06-chip-workload-analysis/ad-robotics-chip-architecture-summary.md) 要把两类 workload 放回统一芯片需求视角的原因。对 archax，这意味着"具身 AI 芯片"不该建模成单一 workload，而应建模成"共享前端 + 领域专用后端"的可配置剖面族，并把两领域的快系统频率（10 Hz vs 200 Hz 量级）作为决定并发隔离设计的关键参数。

## Workload Characterization

按"共享前端 vs 分叉后端"的视角刻画两领域差异。

计算密度：两者前端都含视觉主干 + 时序模型，可共享；后端自动驾驶更重 BEV/occupancy 空间构建，机器人更重 VLA backbone decode + 高频 action head（按控制频率放大）。自动驾驶单帧感知 FLOPs 大但频率低（~10 Hz），机器人 action head 单次小但频率高（100-200 Hz），总负载结构不同。

访存模式：自动驾驶多路相机（6-12 路）+ LiDAR 同步、BEV cache 容量显著；机器人传感器路数少但 proprioception 维度高、KV cache + action chunk + 短时历史显著；两者前端的多模态 token 拼接可共享通路。

并行性：自动驾驶可按 camera / BEV cell / task query head 并行；机器人可按 sensor / action branch 并行，但高频闭环的串行时间步约束更强，快系统的延迟尾巴更敏感。

数据复用：自动驾驶复用 BEV scene feature 给多 query head；机器人复用视觉语言表征和短时本体历史给多动作分支；两侧 world model 的 action-conditioned latent 都可跨候选复用。

量化敏感度：两侧语义主干都可量化；分叉在安全相关输出——自动驾驶的 collision margin / 可行驶区域，机器人的接触力 / 末端姿态 / 平衡，都对量化误差敏感且容错窗口不同（机器人更瞬时），需领域级验证。

瓶颈类型：自动驾驶车端常受多传感器吞吐 + BEV 构建限制；机器人端侧常受 VLM decode latency + action 高频解码 + 控制稳定性限制；双系统下两侧都有"慢系统吃容量、快系统吃延迟"的叠加，但机器人快慢频率差更大、并发隔离更难。

对硬件的核心需求：共享部分是多模态前端、temporal/KV cache、world-model 通路；分叉部分是自动驾驶的多传感器融合 + 空间表征加速，与机器人的高频低延迟 action decode + 高维本体状态融合 + 瞬时安全监控——详见 [AD/Robotics Chip Architecture Summary](../06-chip-workload-analysis/ad-robotics-chip-architecture-summary.md)。

## 参考来源

- Hu et al., `Planning-oriented Autonomous Driving (UniAD)`, CVPR 2023 / arXiv:2212.10156。成熟度：已落地研究，自动驾驶端到端代表。查证日期：2026-06-07。
- Zitkovich et al., `RT-2`, CoRL 2023 / arXiv:2307.15818。成熟度：已落地研究，机器人 VLA 代表。查证日期：2026-06-07。
- Kim et al., `OpenVLA`, arXiv:2406.09246。成熟度：开源机器人 VLA baseline。查证日期：2026-06-07。
- NVIDIA, `GR00T N1`, arXiv:2503.14734。成熟度：humanoid VLA 双系统（System 1 ~120Hz）。查证日期：2026-06-07。
- Figure AI, `Helix: A Vision-Language-Action Model for Generalist Humanoid Control`, figure.ai 技术报告 2025。成熟度：humanoid 双系统（System 1 ~200Hz，35 DoF）。查证日期：2026-06-07。
