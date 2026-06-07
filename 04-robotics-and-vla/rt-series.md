# RT Series

上级：[Robotics and VLA](README.md)
相关：[VLA Fundamentals](vla-fundamentals.md), [Action Tokenizer](action-tokenizer.md), [OpenVLA](openvla.md), [VLA Workload](../06-chip-workload-analysis/vla-workload.md)

## 这页在回答什么问题

这页回答 Google DeepMind 的 RT 系列为什么是机器人 VLA 的关键演进线，以及每一步在 workload 上付了什么代价。RT-1 证明大规模真实机器人数据能训出通用 policy；RT-2 把动作当 text token、让 web-scale VLM 直接输出控制——这一步换来泛化却让模型重了三个数量级；RT-X 把问题推向跨 embodiment 数据。这页关注"每一代把 workload 改成了什么样"，动作编码细节见 [Action Tokenizer](action-tokenizer.md)。

## 为什么它有效：直觉与类比

RT 系列的整条线，可以看成**一个学徒从'熟练工'升级成'读过万卷书的通才'的过程**，每升一级都换来更广的泛化、也背上更重的脑子。

RT-1 是熟练工：它在大量真实机器人演示上练到对常见操作很熟，但它的"脑子"很小——只有 35M 参数，一个 FiLM-conditioned EfficientNet-B3 出视觉语言 token，TokenLearner 把每帧 81 个 token 压成 8 个，Transformer 输出 11 个动作维度各 256 个 bin 的离散动作。对应到机制：小模型 + token 压缩让它能在机器人上以约 3 Hz 实时闭环（单次推理约 15ms），但它的知识全来自机器人数据本身，没见过实验室之外的世界，碰到没演示过的物体就泛化不动。RT-1 的贡献是证明了"规模化真实机器人数据 + Transformer policy"这条路能从 task-specific 走向 multi-task，但它的天花板就是它见过的数据。

RT-2 是请来一位读过整个互联网的通才当大脑：直接拿一个 web-scale 预训练的 VLM（PaLI-X，最大 55B），把机器人动作表示成它本就会写的整数 text token（动作 bin 映射到词表里 0-1000 的整数 token），让它在做视觉问答的同一套机制下顺手吐出动作。对应到机制：这一步的魔法在于动作和语言共用 next-token prediction，于是 VLM 预训练里那些开放词汇识别和常识推理——"那是个香蕉""把它放进最大的碗"——被免费迁移到了机器人控制上，泛化能力跨了一个台阶。但代价是脑子从 35M 重到了几十 B，整整三个数量级。

这个代价直接砸在 workload 上：RT-1 是个能在机器人本体上跑 3 Hz 的轻量 policy，RT-2-PaLI-X-55B 大到只能放在云端多 TPU 上跑、再联网控制机器人，控制频率掉到 1-3 Hz（较小的 RT-2-PaLM-E-12B 约 5 Hz）。对应到机制：动作 token decode 是串行的，且 55B 模型 batch=1 decode 每 token 都要把巨量权重过一遍——泛化能力和延迟是同一笔账的两面。RT 系列这条线，本质就是在"通才的泛化"和"实时闭环的延迟"之间不断重新议价。

## RT-1：小脑子、能实时的起点

RT-1 把机器人控制建模成 Transformer policy：image history + language instruction → FiLM-EfficientNet-B3（约 16M 参数，输出 81 个视觉语言 token）→ TokenLearner（81 压到 8 个 summary token，这是它能实时的关键）→ Transformer → 11 个离散动作变量（7 个手臂/夹爪、3 个底盘、1 个模式切换），各 256 bin。整模 35M，约 3 Hz 闭环。

它的真正贡献不在架构而在数据规模化的证明：用足够多、足够多样的真实演示，单一模型能覆盖多任务而不退化。但它的动作空间和知识都绑死在特定平台和特定数据上——这正是 RT-2 要用 web 知识捅破的天花板。

## RT-2：动作即 text token，泛化与延迟的换价

RT-2 的关键动作是把 web-scale VLM 引进来，并把 robot action 表示成 text-like token，让同一个模型既能做视觉语言任务、又能输出动作。机制的代价在 [VLA Fundamentals](vla-fundamentals.md) 和 [Action Tokenizer](action-tokenizer.md) 已拆过：泛化来自预训练知识迁移，延迟来自巨大 backbone 的串行 token decode。

workload 上 RT-2 是个分水岭——机器人控制从"本体上跑的轻量 policy network"变成"云端大 VLM + 动作 token decode"。它引入了大模型的全套包袱：几十 B 参数、KV cache、batch=1 的 memory-bandwidth 瓶颈、token decode latency。这一代之后，"机器人 policy 的 workload"实质上变成了"LLM/VLM 推理的 workload"，这是后面 OpenVLA、π0、GR00T 全部继承的起点。

## RT-X 与 Open X-Embodiment：跨本体的数据统一

RT-X 关注跨机器人数据。Open X-Embodiment 把 22 个机器人形态、多机构、多任务的数据标准化成统一格式（约 100 万条轨迹规模），目标是训出能跨 platform 迁移的 policy，并验证了"在混合 embodiment 数据上训练能正向迁移"。对 workload，cross-embodiment 不只是数据问题：不同机器人的输入状态维度、动作维度、相机视角、控制频率都不统一，模型要么用 embodiment-specific 的输入/输出 encoder 适配（GR00T N1 就这么做），要么靠通用 action tokenizer（FAST+）把动作空间对齐——这把"动作空间对齐"变成了训练侧一个真实的数据工程和 workload 成本。

## RT-H：动作层次，不必一步到位

RT-H 用 language motion 在高层 task language 和低层 motor command 之间插一层语义动作描述（"move arm left"这种 mid-level 动作），形成 action hierarchy。它说明 VLA 不必从 instruction 一步硬映射到 motor command，中间可以有可复用、可纠错的动作层——这个"先出语言中间步再出动作"的思路，和 π0.5 的 high-level language step 是同一脉络，也预告了双系统里"慢系统出意图、快系统出动作"的分层。

## 一句话理解

RT 系列把机器人从单任务模仿学习推到 VLA：RT-1 用 35M 小模型证明真实数据可规模化、能 3 Hz 实时；RT-2 请来 55B VLM 把动作当 text token、换来 web 知识泛化却把控制频率压到 1-3 Hz 且只能云端跑；RT-X 把数据推向跨本体统一——这条线就是"泛化越强、脑子越重、闭环越难"的反复议价。

## 演进与未来方向（判断）

以下为判断，锚定 2025-2026 联网核实的真实工作，查证日期：2026-06-07。

RT 系列定义了 VLA 的问题框架，但它自己那种"巨大 backbone 串行吐 text token、只能云端 1-3 Hz"的形态，已经被 2024-2026 的后续工作系统性地修正。三条修正线都直接针对 RT-2 的痛点：其一，**模型尺寸下沉**——OpenVLA 用 7B 把 RT-2 的几十 B 拉回能本地探索的量级，SmolVLA 进一步压到消费级硬件；其二，**动作表示换代**——FAST 用频域 token 把 RT-2 那种逐维 binning 的 token 数压一个量级，π0/GR00T 干脆换 flow-matching action expert 绕开串行 token decode；其三，**架构分频**——Helix、GR00T N1 的慢-快双系统把 RT-2"一个大模型既要懂语义又要高频出动作"的矛盾拆成两个频率不同的子系统。

我的判断是，**RT-2 那种"单一巨型 VLM 串行吐 text 动作 token、云端低频闭环"的形态在 2026 已是历史参照而非部署目标**：它证明了 web 知识能迁移到控制（这个洞见是永久的），但它的 workload 形态正好踩中所有最贵的点（巨 backbone + 串行 decode + 云端往返延迟）。后续真正的产业形态是"中等尺寸 backbone（3-7B）+ 高效 action 表示（FAST 或 flow expert）+ 慢-快分频"。

对架构师，RT 系列的价值是它清楚标出了 VLA workload 的演化轨迹：起点（RT-1）是个能塞进机器人本体的小 policy，中段（RT-2）暴跳成云端 LLM 推理 workload，终点正在收敛回"端侧能跑的中等模型 + 分频系统"。这条轨迹对 archax 的含义是，机器人 VLA 的目标工作点不该按 RT-2 那种 55B 云端模型设，而该按 3-7B backbone batch=1 端侧 decode + 高频 action head 的双工作点设——这正是 06 [VLA Workload](../06-chip-workload-analysis/vla-workload.md) 锚定的剖面。

## Workload Characterization

跨代刻画（RT-1 35M 本体级，RT-2 12B-55B 云端级，RT-X 跨本体训练）。

计算密度：RT-1 轻（35M，单次推理约 15ms 即可 3 Hz），主计算是 EfficientNet 视觉编码；RT-2 重（12B-55B backbone），主计算是 VLM decode，单 token 约 2×参数量 FLOP，action head 远小于视觉语言主干；跨代看，计算密度随 backbone 尺寸跨了三个数量级。

访存模式：RT-1 的 81→8 TokenLearner 压缩显著降低 token 访存，是它能实时的关键设计；RT-2 引入 KV cache、batch=1 权重无复用、image/language/history/action token 联合缓存；RT-X 训练侧多源数据格式标准化带来额外 IO 与预处理成本。

并行性：训练可按 episode/task/robot 并行；推理侧视觉编码并行，动作 token decode 串行（RT-2 的整数 text token 路线）；RT-1 因 token 少、模型小，串行 decode 代价远低于 RT-2。

数据复用：VLM 预训练权重把 web knowledge 复用进控制（RT-2 的核心红利）；Open X-Embodiment 的机器人演示跨任务、跨形态复用，验证了混合 embodiment 训练的正迁移。

量化敏感度：视觉语言 backbone 可低比特；动作 token 边界、连续控制恢复、接触任务对量化敏感，需高精度验证；RT-2 的整数 token 路线对 logits 数值范围敏感。

瓶颈类型：RT-1 本体部署近乎无瓶颈（小模型）；RT-2-55B 部署瓶颈是云端大 VLM 的 batch=1 latency + 联网往返，控制频率掉到 1-3 Hz；RT-X 训练瓶颈是多源机器人数据的 IO 与跨本体对齐。

对硬件的核心需求：RT-1 类只需轻量边缘推理；RT-2 类需要 LLM/VLM 推理通路、KV cache 带宽、token decode 低延迟；跨本体训练需要高吞吐多源数据加载——这条从轻到重再回落的轨迹，正是 [VLA Workload](../06-chip-workload-analysis/vla-workload.md) 要按目标工作点取舍的依据。

## 参考来源

- Brohan et al., `RT-1: Robotics Transformer for Real-World Control at Scale`, RSS 2023 / arXiv:2212.06817。成熟度：已落地研究代表（35M，FiLM-EfficientNet-B3 + TokenLearner，约 3 Hz，11 动作维各 256 bin）。查证日期：2026-06-07。
- Zitkovich et al., `RT-2: Vision-Language-Action Models Transfer Web Knowledge to Robotic Control`, CoRL 2023 / arXiv:2307.15818。成熟度：已落地研究，VLA 范式分水岭（PaLI-X 55B，动作即整数 token，云端 1-3 Hz）。查证日期：2026-06-07。
- Padalkar et al., `Open X-Embodiment: Robotic Learning Datasets and RT-X Models`, ICRA 2024 / arXiv:2310.08864。成熟度：已落地，跨本体数据标准化（22 形态、约 100 万轨迹）。查证日期：2026-06-07。
- Belkhale et al., `RT-H: Action Hierarchies Using Language`, RSS 2024 / arXiv:2403.01823。成熟度：研究阶段，语言动作层次。查证日期：2026-06-07。
