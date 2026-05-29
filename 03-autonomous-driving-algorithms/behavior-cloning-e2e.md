# Behavior Cloning E2E

上级：[Autonomous Driving Algorithms](README.md)
相关：[E2E Workload](../06-chip-workload-analysis/e2e-workload.md), [Video Understanding](../02-vision-and-3d-perception/video-understanding.md)

## 这页在回答什么问题

这页回答 Behavior Cloning 式端到端自动驾驶是什么、为什么重新受到关注，以及它和 planning-oriented E2E 的差别。这里的重点不是把端到端等同于一个黑盒网络，而是看它如何把 perception、prediction、planning 和 control 的接口压缩为从 observation 到 action 的学习问题。

## 基本形式

Behavior Cloning 的训练目标是模仿人类或专家系统的驾驶行为：

```text
multi-camera / lidar / route / ego state
   ->
scene encoder + temporal model
   ->
trajectory / waypoint / control token
   ->
imitation loss
```

早期路线以 camera image 到 steering/throttle/brake 为主，代表工作包括 NVIDIA PilotNet 和 ChauffeurNet。新一代路线通常不再直接输出低维控制，而是输出 future waypoints、ego trajectory、action token 或可被低层控制器消费的规划结果。

## 为什么又重要

Behavior Cloning 的优势是训练接口简单、数据规模可扩展，并且能把难以手写的驾驶习惯、交互让行、微妙路权判断纳入学习目标。它适合与大规模 fleet data、自动标注、仿真重采样和 VLA policy 结合。

主要风险是 covariate shift。模型只在专家轨迹附近训练，一旦上线时偏离训练分布，后续 observation 也会偏离，错误可能逐步放大。因此现代系统通常结合 closed-loop evaluation、trajectory perturbation、safety filter、rule prior 或 diffusion/action sampling。

## 输出接口

Behavior Cloning E2E 的输出接口大致有四类：

| 输出形式 | 特点 | workload 影响 |
| --- | --- | --- |
| 控制量 | latency 低，但可解释性弱 | decoder 很轻，主要压力在 encoder |
| waypoint / trajectory | 当前更常见，便于控制和评估 | trajectory head 小，但需要时序稳定 |
| cost / occupancy guided action | 与传统 planner 结合 | 需要额外 cost map 或 future state |
| action token | 适合 VLA / sequence model | decoder 变成 autoregressive 或 hybrid token 生成 |

## 与 planning-oriented E2E 的差别

Planning-oriented E2E 通常保留检测、map、motion、occupancy 等中间监督，让规划 head 成为统一任务之一。Behavior Cloning 则更强调最终行为模仿，可以弱化中间任务，甚至只用 action loss 和少量 safety auxiliary loss。

这不是二选一。工程系统常用中间任务稳定表征，再用 imitation/action loss 把 representation 拉向驾驶决策。

## 成熟度判断

- 成熟：trajectory / waypoint imitation，已广泛用于研究和工程原型。
- 发展中：大规模多模态 Behavior Cloning 与 closed-loop training 的结合。
- 前沿：action tokenizer、VLA policy、diffusion policy 在开放场景驾驶中的可靠部署。

## 一句话理解

Behavior Cloning E2E 把自动驾驶从“手写模块接口”推向“从驾驶数据中学习行为接口”，但它的硬问题从感知精度转移到了分布偏移、闭环稳定性和长尾恢复。

## Workload Characterization

- 计算密度：多摄像头 encoder 和 temporal attention / recurrent state 是主要计算来源；trajectory head 本身通常不重。
- 访存模式：video frame buffer、multi-view feature、ego state、route token 和历史 latent 需要持续缓存；长历史会放大带宽和容量压力。
- 并行性：camera view、frame、feature stage 可并行；autoregressive action token 或 closed-loop rollout 会引入串行依赖。
- 数据复用：同一 scene feature 可复用给 imitation loss、auxiliary perception loss、safety loss 和 uncertainty head。
- 量化敏感度：image encoder 较适合量化；action decoder、trajectory regression、small-object/traffic-light 相关特征需要更谨慎。
- 瓶颈类型：训练侧多为数据吞吐和 video IO 瓶颈；车端推理常由 multi-camera encoder + temporal cache 决定 latency。
- 对硬件的核心需求：稳定多路视频输入、历史状态缓存、低延迟 trajectory decode、batch=1 推理效率、异常帧下的 deterministic fallback。

## 参考来源

- Bojarski et al., `End to End Learning for Self-Driving Cars`, arXiv:1604.07316，成熟度：早期范式，查证日期：2026-05-29。
- Bansal et al., `ChauffeurNet: Learning to Drive by Imitating the Best and Synthesizing the Worst`, RSS 2019 / arXiv:1812.03079，成熟度：经典闭环 imitation baseline，查证日期：2026-05-29。
- Wu et al., `Trajectory-guided Control Prediction for End-to-end Autonomous Driving: A Simple yet Strong Baseline`, NeurIPS 2022 / arXiv:2206.08129，成熟度：研究成熟，查证日期：2026-05-29。
- Chen et al., `Learning from All Vehicles`, CVPR 2022，成熟度：大规模数据学习方向，查证日期：2026-05-29。
