# VLA Fundamentals

上级：[Robotics and VLA](README.md)
相关：[VLA Workload](../06-chip-workload-analysis/vla-workload.md), [Action Tokenizer](action-tokenizer.md), [VLM and VLA for Autonomous Driving](../03-autonomous-driving-algorithms/vlm-vla-for-ad.md), [Attention and Transformer](../01-foundation-model-components/attention-and-transformer.md)

## 这页在回答什么问题

这页回答 VLA（vision-language-action）作为一个范式，到底把"动作"放进了模型的哪一层，以及这个放置位置怎样改写了 workload 的性格。重点不是"机器人能听懂自然语言"，而是 action 从一个外挂控制器变成了模型输出空间的一部分——这一步同时带来了泛化能力和一组全新的实时约束。具体动作如何编码留给 [Action Tokenizer](action-tokenizer.md)，端侧资源分解留给 [VLA Workload](../06-chip-workload-analysis/vla-workload.md)，这页负责把范式本身讲到能写进架构模型的程度。

## 为什么它有效：直觉与类比

VLA 的直觉是**把一个见多识广但从没碰过机器人的人，请来当机械臂的大脑**。这个人在互联网上看过几十亿张图、读过几乎所有能读的文字，所以你跟他说"把桌上那个没喝完的红牛递给我"，他不需要训练就知道红牛长什么样、"没喝完"意味着不能倒、"递给我"意味着末端要朝向你——这些语义和常识是他从 web-scale 预训练里带来的。对应到机制：VLA 复用一个已经在海量图文上预训练好的 VLM 作为 backbone，它的视觉 encoder 和 language model 里已经压进了开放词汇的物体识别和常识推理先验，机器人训练只需要在这个先验之上学"怎么动手"，而不是从零学认识世界。这正是 VLA 比传统 imitation learning policy 泛化强的根源——后者只见过实验室那几百条 demonstration，前者站在互联网知识的肩膀上。

但"会看会想"不等于"会动手"，这里有个关键的衔接动作。传统做法是 perception、planning、control 三段流水线，每段单独训、用手工接口拼起来；VLA 的做法是**让动作和视觉语义共享同一套 token 交互**，把 joint delta、gripper、end-effector pose 这些控制量也表示成模型能生成的东西（离散 token 或被一个 action expert 连续生成），让它们和 vision/language token 在同一个 attention 里对齐。对应到机制：这意味着"看到红牛在右前方"和"末端应该往右前方移动"不再是两个模块靠接口传递，而是在同一个 Transformer 的隐藏状态里被联合表示——动作继承了大模型表示的泛化能力，这是把 action 拉进输出空间换来的核心好处。

代价也由这个衔接而来，而且很硬。机器人是闭环的：模型这一帧输出的动作会立刻改变下一帧的 observation，误差不会被平均掉而是会累积漂移。对应到机制，这要求 VLA 在几十毫秒到数百毫秒内出一个动作（控制频率几 Hz 到几百 Hz 不等），而一个 7B backbone 的 autoregressive decode 天然慢——这就是 VLA 全部 workload 张力的来源：你想要大模型的泛化，又被机器人的实时闭环死死按住延迟。后面所有设计（action chunk、flow matching action expert、慢-快双系统）都是在这对矛盾里找折中。

## 动作进入输出空间的三种位置

把"动作放在哪"说清楚，VLA 的 workload 性格就清楚了，因为不同位置的计算图截然不同。

第一种是**动作即 text-like token**：把每个动作维度量化成离散 bin，当成词表里的特殊 token，让 VLM 像续写句子一样 autoregressive 地吐出动作 token。RT-2 和 OpenVLA 走这条路（见 [RT Series](rt-series.md)、[OpenVLA](openvla.md)）。机制上它最简单——不改 backbone，复用语言模型的 next-token prediction——但代价是动作生成退化成串行 decode，7 个维度就是 7 步串行，控制频率被 token latency 卡死。

第二种是**外挂一个 action expert 做连续生成**：backbone 出多模态隐藏状态，一个小的 diffusion/flow-matching head 以这些状态为条件，一次性生成未来一整段连续动作（action chunk）。π0（PaliGemma 3B VLM + 300M action expert，flow matching 一次出 H=50 步动作）和 GR00T N1（Eagle-2 VLM + DiT action head，一次出 16 步）是代表。机制上它避开了离散量化损失，且一次推理覆盖多个控制周期，但 action expert 的迭代去噪本身是额外计算。

第三种是**显式语言中间步 + 动作**：模型先用语言对自己说一句高层意图（"先打开抽屉"），再据此生成 motor command。π0.5 就是先产出语言形式的 high-level action，再用 flow matching action expert 出低层动作。机制上语言中间步给了可解释性和更强的长程任务分解，但多了一段语言 decode，延迟更高——这也是下文"范式演进"要讨论的、正在被压缩的环节。

## 一句话理解

VLA 把机器人动作从外挂控制器拉进多模态大模型的输出空间，让动作继承 web-scale 预训练的语义泛化；代价是 workload 被钉在"大模型 backbone + 闭环实时控制"这对矛盾上——visual token 输入大、backbone decode 慢，却要在几十毫秒到数百毫秒内出动作，所有设计都是在这对矛盾里折中。

## 演进与未来方向（判断）

以下为判断，锚定 2025-2026 联网核实的真实工作，查证日期：2026-06-07。

主线很清晰：**从"显式语言中间步 → 直接 visual-to-action"，再到"慢-快双系统把两者分频"**。早期 RT-2 把动作当 text token 串行吐，π0.5 一度强调先出语言 high-level step 再出动作，因为语言分解帮长程任务。但语言 token 既慢又会幻觉，所以最有产业动量的形态正在收敛到双系统：一个大 VLM（System 2）在低频上做语义理解和任务规划，一个轻量 action policy（System 1）在高频上做连续运动生成。Figure Helix 是最直白的例证——System 2 是 7B VLM 跑在 7-9 Hz，System 1 是 reactive visuomotor policy 跑在 200 Hz，控制整个上半身 35 个自由度；NVIDIA GR00T N1 同构，System 2（Eagle-2 VLM）约 10 Hz、System 1（DiT）约 120 Hz。这个分频不是工程妥协而是范式判断：语义推理和实时控制对算力的需求性格根本不同，硬把它们塞进同一个频率要么慢得不能闭环、要么蠢得不懂语义。

我的几条判断。其一，**双系统会是未来 2-3 年通用机器人 VLA 的主流形态**，因为它精确对应了上面那对"泛化 vs 实时"的矛盾——这一点和自动驾驶的慢-快双系统同源（见 [VLM and VLA for Autonomous Driving](../03-autonomous-driving-algorithms/vlm-vla-for-ad.md)），两个领域独立收敛到同一架构，说明它是结构性的解而非巧合。其二，**离散 action token 路线会被连续 action expert（flow/diffusion）和高效 tokenizer（FAST）两头挤压**：前者避开量化损失、后者把 token 数压一个量级，纯 binning 的串行 decode 在高频控制上越来越没竞争力。其三，**模型尺寸在分化**——研究前沿往大走（人形通用模型），但部署前沿往小走（SmolVLA 这类把 VLA 压到能在消费级硬件上跑的工作），两端对硬件提的需求完全不同。

对架构师，这条演进的 workload 含义是硬的：双系统意味着一颗芯片要**同时承载两个工作点**——一个 batch=1、memory-bandwidth 主导、KV cache 吃容量的大 VLM decode 流，和一个高频、低延迟、确定性强的 action policy 流，两者还要并发隔离不互相抢资源。这正是 06 [VLA Workload](../06-chip-workload-analysis/vla-workload.md) 的核心。对 archax，关键是把"慢系统工作点"和"快系统工作点"建模成两个独立但共存的负载剖面，并在 Interaction 轴上显式表达它们对 KV cache 带宽、片上 buffer 的竞争——按平均值评估会同时低估快系统的延迟尾巴和慢系统的容量峰值。

## Workload Characterization

以一个 3B-7B backbone、单臂或上半身 humanoid、batch=1 端侧部署为基准刻画。

计算密度：VLM/LLM backbone 是绝对主计算，一个 7B backbone 单 token decode 约 14 GFLOP（2×参数量）量级，而 action head（离散 token 的 logits 投影，或 300M 级 action expert）小一个数量级以上；但 action head 按控制频率高频执行（几十到几百 Hz），其总负载占比会被频率放大，不能按单次 FLOPs 忽略。

访存模式：batch=1 decode 下权重无法在 batch 维复用，每出一个 token 要把整套 backbone 权重从 HBM 过一遍，是 memory-bandwidth-bound 的典型；visual token（单相机 224px 经 SigLIP/DINOv2 约 256 个 token，多相机线性叠加）+ language token + robot state + KV cache 共占带宽与容量；多相机和 wrist camera 直接抬高 prefill 的 token 数。

并行性：vision encoder 对多相机输入可并行；backbone 的 prefill 阶段 token 维可并行，但 action 的 autoregressive decode（离散 token 路线）受因果依赖串行；action chunk 和 flow-matching 一次出 H 步动作，把控制频率从"每动作一次推理"降到"每 chunk 一次推理"，是缓解串行性的关键手段。

数据复用：同一份多模态隐藏状态可同时服务动作生成、任务完成判断、失败检测、语言解释多个 head；短任务内 instruction 和场景 context 的 KV 可跨控制周期复用，但视觉 token 每帧更新、复用窗口短。

量化敏感度：backbone 权重可探索 INT8/INT4（SmolVLA 类工作即靠此压部署成本）；但 action 边界相关的输出对量化更敏感——离散 token 的 logits、连续动作恢复、gripper 开闭和接触阶段的动作量化误差会直接变成控制抖动，需任务级验证而非按 backbone 标准放过。

瓶颈类型：端侧几乎总是 latency + memory-capacity + KV-cache-bandwidth 三者主导，batch=1 让算力利用率天然低；双系统部署下，慢系统吃容量、快系统吃延迟尾巴，最坏闭环延迟由两者叠加决定；云端训练侧瓶颈是多源机器人数据 IO 与跨 embodiment 对齐。

对硬件的核心需求：低 batch VLM 推理的高带宽通路、KV cache 的容量与带宽管理、多传感器 token 的拼接与 prefill、action head（离散 logits 或迭代去噪）的低延迟输出，以及慢-快两个工作流的并发隔离——详见 [VLA Workload](../06-chip-workload-analysis/vla-workload.md)。

## 参考来源

- Zitkovich et al., `RT-2: Vision-Language-Action Models Transfer Web Knowledge to Robotic Control`, CoRL 2023 / arXiv:2307.15818。成熟度：已落地研究范式，动作即 text token 的代表。查证日期：2026-06-07。
- Kim et al., `OpenVLA: An Open-Source Vision-Language-Action Model`, arXiv:2406.09246。成熟度：开源研究 baseline。查证日期：2026-06-07。
- Black et al. (Physical Intelligence), `π0: A Vision-Language-Action Flow Model for General Robot Control`, arXiv:2410.24164。成熟度：研究阶段，flow-matching action expert（PaliGemma 3B + 300M expert，H=50）。查证日期：2026-06-07。
- Physical Intelligence, `π0.5: a Vision-Language-Action Model with Open-World Generalization`, arXiv:2504.16054。成熟度：研究阶段，显式语言 high-level step + flow action expert。查证日期：2026-06-07。
- Figure AI, `Helix: A Vision-Language-Action Model for Generalist Humanoid Control`, figure.ai 技术报告 2025。成熟度：产业研究原型，慢-快双系统（System 2 7B @7-9Hz，System 1 @200Hz，35 DoF）。查证日期：2026-06-07。
- NVIDIA, `GR00T N1: An Open Foundation Model for Generalist Humanoid Robots`, arXiv:2503.14734。成熟度：产业研究原型，双系统（VLM @10Hz，DiT @120Hz）。查证日期：2026-06-07。
