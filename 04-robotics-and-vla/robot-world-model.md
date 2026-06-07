# Robot World Model

上级：[Robotics and VLA](README.md)
相关：[VLA Fundamentals](vla-fundamentals.md), [GR00T](groot.md), [World Model Fundamentals](../05-world-model-and-generative/world-model-fundamentals.md), [World Model Workload](../06-chip-workload-analysis/world-model-workload.md)

## 这页在回答什么问题

这页回答机器人 World Model 和 VLA 是什么关系，以及它把 workload 推向了哪里。VLA 是反应式的 policy（observation → action）；World Model 多问一句"如果我这么动，接下来会怎样"，在行动前预测未来状态。重点是 action-conditioned 的 latent rollout——这是它区别于纯视频生成的核心，也是它 workload 性格的来源。世界模型的通用机制见 [World Model Fundamentals](../05-world-model-and-generative/world-model-fundamentals.md)，资源分解见 [World Model Workload](../06-chip-workload-analysis/world-model-workload.md)。

## 为什么它有效：直觉与类比

机器人 World Model 的直觉是**让机器人在脑子里先排练一遍再动手**。一个熟练的人去拿叠在一起的杯子，会先在脑中预演"如果我从这个角度抓，上面那个会不会倒"，预演到不会倒的方案才真去抓。对应到机制：World Model 学一个 action-conditioned 的预测器，给定当前 latent state 和一个候选 action，预测下一时刻的 latent state——把这个预测沿时间迭代若干步，就是在 latent 空间里"排练"一段未来。policy 可以对多个候选动作各排练一遍，挑后果最好的执行。这正是 V-JEPA 2-AC 的做法：它学一个 action-conditioned predictor 表达"如果做这个动作、就会变成那个状态"的因果，再用 model-predictive control（MPC）在 latent 空间规划，零样本控制 Franka 机械臂完成 image-goal 的抓放。

为什么排练要在 latent 空间而不是预测像素？因为机器人要的是"能用于决策的预测"，不是"好看的视频"。对应到机制：预测每一个像素既贵又抓错了重点——决策只需要知道"杯子会不会倒、夹爪有没有抓住接触点",这些是状态量，不是纹理细节。在压缩的 latent 空间里 rollout，省掉了像素解码的巨大开销，且让模型把容量花在物理相关的状态演化上。这是 robot world model 和"视频生成模型"的根本分界：漂亮视频不等于可控物理预测，对机器人更关键的是 action-conditioned 的 latent dynamics、接触结果和物体状态。FLARE 把这点做到极致——它甚至不显式生成未来观测，而是给标准 VLA 加几个 "future token"，训练时让这些 token 的特征对齐未来观测的 latent embedding（隐式 world modeling），作为 action flow-matching 之外的辅助损失，仅靠极小架构改动就让 policy"顺带"学会预判后果，多任务上比基线提升最多约 26%。

机器人场景和自动驾驶的 world model 关注点不同，这决定了建模对象。对应到机制：自动驾驶的 world model 重在大范围交通参与者和几何空间的演化（见 05 的 BEV/occupancy world model）；机器人重在近场接触、可操作物体、手眼协调，以及"动作导致的局部状态变化"——同样是 action-conditioned dynamics，机器人的 action 维度更高、接触更非线性、horizon 更短但更精细。这让机器人 world model 的 latent 要编码接触和物体姿态，而非交通流。

## World Model 怎么接进机器人系统

机器人 World Model 可以预测三类东西，对应三种用法。预测 latent / belief state，给 policy 提供可规划的紧凑表示（V-JEPA 2-AC、FLARE 这条线）；预测 visual future（未来图像/视频），用于仿真、数据生成、失败回放（GR00T-Dreams 这类合成数据管线）；预测 physical outcome（物体 pose、接触、成败），直接做动作评估和风险判断。

它和 VLA 的结合有三种方式，每种 workload 含义不同。一是 policy pretraining：用海量视频/交互数据学可迁移的物理表征，再迁到 policy（V-JEPA 2 先在 100 万+小时互联网视频上自监督预训练，再用约 62 小时无标注机器人视频做 action-conditioned 后训练）——这把绝大部分计算压在云端预训练，部署侧轻。二是 action rollout：部署时对多个候选动作各预测未来、挑风险最低的——这把多候选并发 rollout 的计算压到推理侧实时路径上，是最贵的用法。三是 data generation：生成长尾交互数据补真实演示之不足——这是离线批量生成，吞吐导向而非延迟导向。三种用法对硬件提的需求完全不同，混为一谈会错估 workload。

## 一句话理解

Robot World Model 让机器人在 latent 空间里"先排练再动手"：学一个 action-conditioned 的 latent dynamics 预测器，预演候选动作的后果再决策；它的价值在 latent rollout（而非像素视频），把 workload 从反应式 VLA 推向 action-conditioned rollout、多候选并发评估和生成式预训练——用法不同（预训练 / 实时 rollout / 数据生成）则瓶颈完全不同。

## 演进与未来方向（判断）

以下为判断，锚定 2025-2026 联网核实的真实工作，查证日期：2026-06-07。

2025-2026 这条线有两个清晰演进。其一，**world model 正从"显式生成未来观测"转向"隐式 latent 对齐"**。早期想法是预测未来视频帧（贵且抓错重点），FLARE（CoRL 2025，arXiv:2505.15659）证明只要让 policy 内部的 future token 对齐未来观测 latent 就够——不解码像素、几乎不加参数，却拿到预判收益。V-JEPA 2（arXiv:2506.09985）同样在 latent 空间做 action-conditioned 预测 + MPC 规划，零样本控制真机。这条"隐式/latent world model"路线的 workload 优势巨大：省掉了像素解码这个最重的环节。其二，**world model 作为合成数据引擎正在成为人形 VLA 的数据基础设施**——GR00T-Dreams 类管线用 world model 在仿真里批量生成训练数据，36 小时产一版，补真实人形演示数据极贵之缺（见 [GR00T](groot.md)）。

我的几条判断。其一，**latent / 隐式 world model 会主导机器人这一侧，显式视频生成主要留给数据生成和可视化**，因为决策只需状态量、不需像素，且 latent rollout 才跑得起实时多候选。其二，**实时参与闭环决策（部署时 online rollout 选动作）仍是前沿**——它最贵（多候选 × rollout horizon × 每步预测都在实时路径上），到 2026 多数落地是"world model 训 policy / 生成数据"而非"world model 在线 rollout 决策"。其三，**预训练-后训练的算力分配会越发不对称**：V-JEPA 2 那种"百万小时视频云端预训练 + 几十小时机器人数据后训练"的模式，把计算极度压向云端预训练侧，部署侧只留轻量 MPC——这是个对硬件分工很明确的信号。

对架构师，机器人 world model 的 workload 含义按用法分裂，且正是 06 [World Model Workload](../06-chip-workload-analysis/world-model-workload.md) 要刻画的：预训练侧是视频/latent 模型的大吞吐训练（compute + 数据 IO bound）；实时 rollout 侧是多候选并发的 action-conditioned latent dynamics（多个 rollout 流并发、每流沿时间迭代、horizon 内有依赖——这是 [Mamba and SSM](../01-foundation-model-components/mamba-and-ssm.md) 末尾预判的"状态递推"类算子的机器人版）；数据生成侧是离线批量吞吐。对 archax，最该显式建模的是"实时 rollout"这种"批并行（多候选）× 迭代深度（horizon）× 状态递推"的复合负载，它既不是纯 LLM decode 也不是纯 GEMM。

## Workload Characterization

按三种用法分别刻画（预训练 / 实时 rollout / 数据生成）。

计算密度：预训练侧是视频/latent 模型（Diffusion 或 Transformer/JEPA 类）的重训练，compute-bound；实时 rollout 侧的密度 = 候选动作数 × rollout horizon × 每步 latent dynamics 预测，多候选会成倍放大；数据生成侧是离线大批量生成，吞吐导向。

访存模式：历史观测、candidate action、object/latent state、rollout 中间帧需缓存；latent rollout（V-JEPA/FLARE 类）省掉像素解码的大张量往返，是相对视频生成 world model 的根本省法；预训练侧是海量视频数据的高带宽 IO。

并行性：不同 action candidate、不同 sample、不同 scene 之间完全可并行（天然 batch 轴）；单条 rollout 内部沿时间步有依赖（状态递推，不可并行展开）——这是"批并行强、迭代深度不可省"的结构，和 SSM 的递推同构。

数据复用：同一份 observation encoding 复用于多个候选动作的 rollout 起点；同一份生成数据可同时用于 policy training 和 failure replay；预训练学到的物理表征跨任务、跨 embodiment 复用（V-JEPA 2 的核心红利）。

量化敏感度：latent dynamics 主体可尝试低比特；但 contact boundary、collision、grasp success 等决策相关输出对误差敏感（误差会沿 rollout horizon 累积，类似 SSM 状态递推的误差累积），需保守；隐式 latent 对齐（FLARE）对辅助损失的数值稳定性敏感。

瓶颈类型：预训练受数据/显存/生成模型吞吐限制；实时 rollout 受候选数 × horizon × 单步延迟限制（最贵、最受实时约束）；数据生成受批量吞吐限制。三者不可混评。

对硬件的核心需求：生成/latent 预测模型的加速、多候选并发 rollout（批并行 + 沿时间迭代的状态递推支持，与 [Mamba and SSM](../01-foundation-model-components/mamba-and-ssm.md) 的 scan/state 需求同源）、历史 latent 缓存、与 policy decoder 共享表征、预训练侧的大视频数据吞吐——详见 [World Model Workload](../06-chip-workload-analysis/world-model-workload.md)。

## 参考来源

- Ha and Schmidhuber, `World Models`, NeurIPS 2018 / arXiv:1803.10122。成熟度：已落地经典概念，latent rollout 的源头。查证日期：2026-06-07。
- Hafner et al., `Mastering Diverse Domains through World Models (DreamerV3)`, arXiv:2301.04104。成熟度：已落地研究路线，latent world model + planning。查证日期：2026-06-07。
- Chi et al., `Diffusion Policy: Visuomotor Policy Learning via Action Diffusion`, RSS 2023 / arXiv:2303.04137。成熟度：已落地，机器人动作生成成熟研究。查证日期：2026-06-07。
- Zheng et al., `FLARE: Robot Learning with Implicit World Modeling`, CoRL 2025 / arXiv:2505.15659。成熟度：2025 研究阶段，future-token 隐式 latent 对齐（多任务最高 +26%）。查证日期：2026-06-07。
- Assran et al. (Meta), `V-JEPA 2: Self-Supervised Video Models Enable Understanding, Prediction and Planning`, arXiv:2506.09985。成熟度：2025 研究阶段，action-conditioned latent world model（百万+小时视频预训练 + 约 62 小时机器人后训练，MPC 零样本控制 Franka）。查证日期：2026-06-07。
