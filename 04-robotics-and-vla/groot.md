# GR00T

上级：[Robotics and VLA](README.md)
相关：[VLA Fundamentals](vla-fundamentals.md), [Action Tokenizer](action-tokenizer.md), [Robot World Model](robot-world-model.md), [VLA Workload](../06-chip-workload-analysis/vla-workload.md)

## 这页在回答什么问题

这页拆解 NVIDIA GR00T N1 的定位与 workload。它是面向 generalist humanoid robots 的开放基础模型，关键不在"换到人形机器人上"，而在它把"语义推理"和"高频运动控制"显式拆成两个频率不同的子系统（dual-system），用一个慢 VLM + 一个快 diffusion action head 解决"大模型懂语义但慢、控制要高频"的矛盾。这是机器人 VLA 双系统范式最完整的开源样本。

## 为什么它有效：直觉与类比

GR00T N1 的核心设计直觉是**一个深思熟虑的项目经理 + 一个手脚麻利的技工，各自按自己的节奏工作**。项目经理（System 2）看现场、读指令、想清楚"现在该干哪一步"，但他想得慢、几秒看一次全局就够；技工（System 1）不操心大局，只盯着项目经理给的当前意图，手上以快得多的节奏连续微调动作，让机械臂平稳地动起来。对应到机制：System 2 是一个预训练 VLM（Eagle-2），约 10 Hz 处理视觉和语言、输出表征当前任务意图的 token；System 1 是一个用 flow-matching 训练的 Diffusion Transformer（DiT），约 120 Hz，cross-attend 到 System 2 的输出 token，迭代去噪生成连续 motor action。两套频率差一个数量级，正好对应"想"和"做"两件事天然不同的节奏。

为什么非要拆成两个频率？因为硬把它们合成一个会两头不讨好。对应到机制：让 10 Hz 的大 VLM 直接出每一帧 120 Hz 的动作，它根本来不及，机器人会卡顿；让 120 Hz 的小 policy 自己去理解开放语言指令，它没那个语义容量，碰到没见过的任务就懵。拆开后，慢系统的语义泛化（来自 VLM 的 web-scale 预训练）和快系统的实时连续控制各得其所——这正是 Figure Helix（System 2 7B @7-9Hz、System 1 @200Hz）独立收敛到的同一结构，两家不约而同说明这是人形高自由度控制的结构性解。

人形机器人把这个矛盾放大了，所以双系统在这里尤其必要。对应到机制：相比单臂桌面操作，humanoid 多了动力学、平衡、双臂协调、躯干和手指——Helix 控制上半身 35 个自由度。动作维度从 7 暴涨到几十，且这些维度强耦合（重心、接触、平衡）；如果让慢 VLM 直接回归这么高维、这么高频的动作，既慢又不稳。System 1 用 flow-matching 一次出 16 步动作 chunk、迭代去噪连续生成，既能表达高维强耦合动作、又能高频闭环——这是 GR00T 比单臂 VLA 在 workload 上复杂得多的根本原因：不是模型更大，而是动作通道更多、控制层级更深、快慢两流必须并发。

## 真实结构与数字锚点

公开的 GR00T-N1-2B 总参数 2.2B，其中 VLM（Eagle-2）约 1.34B，其余是 DiT action head 和 encoder/decoder。System 1 的 DiT 用 adaptive layer normalization 注入去噪步条件，交替 cross-attention（条件于 VLM 的视觉语言 token）和 self-attention（作用于加噪的 action token + state embedding）块，一次预测 16 步动作 chunk。频率上 System 2 约 10 Hz、System 1 约 120 Hz（L40 GPU 上）；整模采样 16 个动作约 63.9ms。跨 embodiment 靠 embodiment-specific 的 state/action encoder-decoder 适配不同机器人的状态和动作维度——这让同一个 backbone 能从桌面机械臂迁到人形。

训练数据是真实机器人数据 + 人类视频 + 仿真合成数据的混合，且强调用 GR00T-Dreams 这类 world-model 蓝图生成合成数据来补真实数据之不足（与 [Robot World Model](robot-world-model.md) 直接相关）。GR00T N1.5（约 3B+，Eagle-2.5 VLM 约 2.1B）是其升级，VLM grounding 更强（GR-1 grounding benchmark 的 IoU 从 35.5 升到 40.4），后续还有 N1.6/N1.7 等迭代——这些主要以技术报告、研究网站和 HuggingFace 权重发布，不都对应独立 arXiv 论文。

## 一句话理解

GR00T N1 是机器人 VLA 双系统范式最完整的开源样本：约 1.34B 的 Eagle-2 VLM（System 2）以 10 Hz 出语义意图，一个 flow-matching DiT（System 1）以 120 Hz cross-attend 这些意图、迭代去噪出 16 步连续动作；它把 workload 从单臂的"一个大模型串行出动作"改成"慢 VLM + 快 DiT 两流并发"，这正是高自由度人形控制的结构性解。

## 演进与未来方向（判断）

以下为判断，锚定 2025-2026 联网核实的真实工作，查证日期：2026-06-07。

GR00T 这条线的演进是**双系统范式的快速迭代 + world-model 合成数据闭环**。从 N1（2.2B，Eagle-2）到 N1.5（3B+，Eagle-2.5，grounding 更强）再到 N1.6/N1.7，主要变化集中在三处：VLM backbone 升级（更强的物理 grounding）、合成数据管线（GR00T-Dreams 用 world model 36 小时生成一版训练数据）、以及推理/reasoning 能力增强。同期 Figure Helix 在产业侧独立验证了同一双系统结构（且把 System 1 推到 200 Hz、覆盖 35 DoF 上半身、能双机协作）。两条线共同说明：**慢-快双系统已经从"一种架构选择"变成人形通用控制的事实标准**。

我的几条判断。其一，**双系统的快慢分频会进一步固化，但两系统的接口（慢系统给快系统传什么）会成为竞争焦点**——传 token 还是传 latent、传多频繁、快系统对慢系统输出的依赖有多紧，直接决定 workload 的并发结构。其二，**合成数据闭环（world model 生成训练数据）会成为人形 VLA 的标配**，因为真实人形演示数据极贵、embodiment 又分散，靠 world model 在仿真里批量生成是唯一规模化的路（这把 [Robot World Model](robot-world-model.md) 从"可选增强"变成"数据供给基础设施"）。其三，**人形 VLA 的部署仍处在"仿真/数据/少量任务成熟、开放环境长程自主仍前沿"的阶段**——公开模型不等于量产验证，这个判断到 2026 仍成立。

对架构师，GR00T 类双系统给出的 workload 含义最硬，且正是 06 [VLA Workload](../06-chip-workload-analysis/vla-workload.md) 的核心场景：一颗芯片要同时承载**两个性格相反的工作点**——System 2 是 batch=1、~10 Hz、memory-bandwidth 与 KV cache 主导的大 VLM；System 1 是 ~120 Hz、低延迟、固定去噪迭代、动作通道多的小 DiT。两者频率差一个数量级却要并发隔离、还要在每个 System 2 周期把意图同步给 System 1。对 archax，这必须建模成两个共存的负载剖面 + 一个跨频同步的 Interaction，按平均算力会同时低估 System 1 的高频延迟尾巴和 System 2 的容量峰值；而合成数据闭环则把 world model 推理（见 [Robot World Model](robot-world-model.md)）也拉进了这条 workload 链的训练侧。

## Workload Characterization

以 GR00T N1（2.2B，Eagle-2 VLM 1.34B + DiT action head，humanoid 多通道，batch=1）为基准。

计算密度：System 2 VLM（约 1.34B）是语义侧主计算，~10 Hz 触发；System 1 DiT 单步不大但按 ~120 Hz × 去噪迭代步数高频执行，总负载被频率放大；humanoid 的高维动作（数十 DoF）让 action head 比单臂 7-DoF 显著重。整模采样 16 动作约 63.9ms（L40）。

访存模式：多相机视觉 token + robot proprioception + 任务 token + 历史状态需持续缓存；System 1 高频读 System 2 输出的 cross-attention 条件 token + action/state 缓存；DiT 去噪的中间张量反复读写；仿真训练侧合成数据吞吐巨大。

并行性：传感器编码、多动作分支、仿真场景可并行；System 1 的 chunk 内 16 步可一次生成、但去噪 step 间有迭代依赖；快慢两流并发但单机器人闭环受时间步约束。两系统频率差一个数量级，是并发隔离的主难点。

数据复用：共享的 embodied 表征复用于双臂、抓取、导航、任务判断多个分支；System 2 输出的意图 token 在一个 ~10 Hz 周期内被 System 1 的多个 120 Hz 步复用（约 12 次）；world-model 生成的合成数据回流训练。

量化敏感度：语义 VLM 可量化；但平衡、接触、手指控制、躯干协调相关的高维动作输出对量化误差敏感（误差直接威胁稳定与安全），DiT 去噪的数值稳定性也需保守处理。

瓶颈类型：训练侧是仿真/真实/合成数据吞吐与模型规模；部署侧是双系统并发——System 2 吃容量与带宽、System 1 吃高频延迟，最坏闭环延迟由两者叠加 + 跨频同步开销决定；多传感器 IO 也是端侧约束。

对硬件的核心需求：高效低 batch VLM 推理（System 2）、低延迟高频迭代去噪（System 1 DiT）、多流传感器输入、多控制通道同步、快慢两工作流的并发隔离与跨频同步，以及训练侧的仿真/合成数据集群吞吐——详见 [VLA Workload](../06-chip-workload-analysis/vla-workload.md)。

## 参考来源

- NVIDIA, `GR00T N1: An Open Foundation Model for Generalist Humanoid Robots`, arXiv:2503.14734。成熟度：2025 产业研究原型（2.2B 总参，Eagle-2 VLM 1.34B + flow-matching DiT，System 2 ~10Hz / System 1 ~120Hz，16 动作 chunk）。查证日期：2026-06-07。
- NVIDIA GEAR, `GR00T N1.5: An Improved Open Foundation Model for Generalist Humanoid Robots`, research.nvidia.com/labs/gear/gr00t-n1_5/ 与 huggingface.co/nvidia/GR00T-N1.5-3B。成熟度：2025 技术报告/权重发布（约 3B+，Eagle-2.5 VLM），无独立 arXiv。查证日期：2026-06-07。
- Figure AI, `Helix: A Vision-Language-Action Model for Generalist Humanoid Control`, figure.ai 技术报告 2025。成熟度：产业研究原型，双系统独立验证（System 2 7B @7-9Hz，System 1 @200Hz，35 DoF 上半身，双机协作）。查证日期：2026-06-07。
- Black et al. (Physical Intelligence), `π0`, arXiv:2410.24164。成熟度：研究阶段，flow-matching action expert 的同源路线。查证日期：2026-06-07。
