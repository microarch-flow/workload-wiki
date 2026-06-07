# Edge-cloud Collaborative World Model

上级：[World Model and Generative Intelligence](README.md)
相关：[Latent World Model](latent-world-model.md), [Video World Model](video-world-model.md), [Cloud Inference and Simulation Chip](../06-chip-workload-analysis/cloud-inference-and-simulation-chip.md), [World Model Workload](../06-chip-workload-analysis/world-model-workload.md)

## 这页在回答什么问题

这页回答 World Model 为什么天然分裂成端、云两套硬件需求，以及这条分裂线该划在哪。重的生成式 world model（视频/4D occupancy）造数据、做仿真、挖长尾，几乎只能在云端；端侧要的是低延迟、可验证、与 planner 紧耦合的轻量 latent/occupancy 预测。两类硬件需求几乎相反——理解这条分裂，才能把 05 章前七篇的表示选择正确映射到 06 的两类芯片。

## 为什么它有效：直觉与类比

端云协同的直觉是**飞行员的训练分两段：地面的全任务模拟器和座舱里的实时仪表**。模拟器又大又贵、跑在地面机房，能把暴风、引擎失效、罕见进近反复造出来练；但真飞起来，飞行员靠的是座舱里低延迟、确定、断网也能用的仪表与判断，绝不会每个操纵都打电话问地面。World Model 是同一结构：云端是"全任务模拟器"，端侧是"座舱仪表"。对应到机制，云端跑重的生成式 world model（GAIA-2、Cosmos 类）造长尾/counterfactual 数据、做闭环仿真、训练与回放，允许高延迟、大 batch、高功耗；端侧跑轻的 latent/occupancy 预测做短 horizon 风险评估与 candidate scoring，必须低延迟、确定性强、本地自洽。

为什么分裂线不是"把同一个大模型切两半"，而是"按时间尺度分任务"：云端做的是"扩展经验"（离线、可大规模并行、不受实时约束），端侧做的是"实时判断"（在线、闭环、毫秒级）。对应到机制，这两类任务的 workload 性格本就相反——前者是吞吐/容量/IO 问题，后者是延迟/功耗/确定性问题，硬切一个模型只会让两边都不合适。

为什么两边都不能省：不能全放云端，因为自动驾驶/机器人不能依赖网络做实时闭环控制——延迟、连接可靠性、隐私、安全都要求端侧独立运行；不能全放端侧，因为高保真 video world model、长 horizon 仿真、多样本 counterfactual 的成本（前面几篇的三重乘法）端侧扛不住。对应到机制，这正是 05 章表示谱系的两端各归其位：像素生成式归云、latent/occupancy 预测式归端。

## 职责拆分与背后的 workload 性格

| 位置 | 适合任务 | 硬件性格 | 连接 |
| --- | --- | --- | --- |
| 云端训练 | 大 world model 训练、视频生成、仿真、数据闭环 | 显存容量 + 跨卡通信 + 数据吞吐 | [Cloud Inference and Simulation Chip](../06-chip-workload-analysis/cloud-inference-and-simulation-chip.md) |
| 云端评测 | 多版本回放、scenario mining、counterfactual generation | 大 batch 并行 + storage IO | 同上 |
| 端侧推理 | 短 horizon risk、trajectory candidate scoring、fallback | 低延迟 + state cache + 确定性 | [World Model Workload](../06-chip-workload-analysis/world-model-workload.md) |
| 端侧日志 | 失败触发、rare event capture、feature/log 上传 | 低带宽上行 + 触发逻辑 | 闭环回流 |

云端那支是训练/仿真三角（显存容量 + 跨卡 all-reduce + 数据 IO），靠大 batch 把端侧 batch=1 的带宽瓶颈反转成 compute-friendly（见 [Cloud Inference and Simulation Chip](../06-chip-workload-analysis/cloud-inference-and-simulation-chip.md)）。端侧那支是 latent/occupancy 的带状态迭代 rollout——`encode once → rollout many`、state cache 常驻、单条 rollout 沿时间串行、多 candidate 并发，和 [Latent World Model](latent-world-model.md) 一致。

## 连接的瓶颈：协同不是免费的

端云协同本身引入一类常被忽略的 workload：数据回流与模型更新的带宽与 IO。车队/机器人日志（失败触发、rare event）上行造数据，云端生成式 world model 把长尾场景造出来喂回训练，更新后的模型再下发端侧。这条回路的瓶颈在 storage IO（云端扫数据湖）、上行带宽（端侧选择性上传而非全量）、以及端侧加载新模型的 PCIe/存储带宽。对架构师，端侧芯片的对外带宽（PCIe、车载以太）和云端的 storage/互连，是端云协同能否高效闭环的隐性约束，不是只看两端各自的算力。

## 一句话理解

World Model 天然分裂成两套相反的硬件需求：云端用容量/吞吐/IO 跑重的生成式 world model"扩展经验"（造数据、仿真、训练），端侧用低延迟/state cache 跑轻的 latent/occupancy 预测"实时判断"；分裂线按时间尺度划，而非切一个模型，且数据回流的 IO/带宽是协同的隐性瓶颈。

## 演进与未来方向（判断）

以下为判断，锚定 2025-2026 真实工作。查证日期：2026-06-07。

第一，**云端 world foundation model 正在成为标准的"数据与仿真工厂"，端云分工日趋固化**。NVIDIA Cosmos（arXiv:2501.03575）把"先在数字孪生世界里训练 policy"做成开源平台，Waymo World Model（2026-02）做闭环仿真，配合 nuPlan、NAVSIM 这类闭环评测基准，形成"云端生成 + 仿真 + 评测 → 端侧部署 → 日志回流"的闭环。我的判断是这条云端线成熟度最高且会持续重投入，因为它不受实时约束、收益直接（治长尾、造 corner case）。

第二，**端侧这支随轻量 latent world model 成熟而真正可行，但形态会是"短 horizon、结构化、可验证"而非完整生成 rollout**。V-JEPA 2-AC 这类无像素 decoder、比 Cosmos 快约 30× 的 latent 预测，正是端侧需要的形态。我的判断是端侧 world model 近期落在"轻量 latent dynamics 做短 horizon 风险评估 + candidate scoring"，与 perception/planning 共享中间表征以摊薄成本。对 archax，端云协同应建模为两个截然不同的工作点而非一个：云端扫"训练吞吐 × 显存容量 × 数据 IO × 跨卡通信"，端侧扫"horizon × candidate × state-cache × latency × 对外带宽"。两套需求几乎相反，分别对应 06 的 [Cloud Inference and Simulation Chip](../06-chip-workload-analysis/cloud-inference-and-simulation-chip.md) 与 [World Model Workload](../06-chip-workload-analysis/world-model-workload.md)；把数据回流的 PCIe/storage 带宽显式建进协同模型，是这条主线对架构探索的额外要求。

## Workload Characterization

计算密度：云端是大 world model 训练/生成/仿真的高吞吐计算（compute + capacity）；端侧是低 batch、短 horizon、轻 latent/occupancy 的低延迟推理。

访存模式：云端需高吞吐数据湖 + 大 activation/KV（capacity/IO 主导）；端侧需 history latent、BEV/occupancy cache、candidate buffer（小张量、常驻 SRAM）。

并行性：云端按 scenario/model/sample 大规模并行（吞吐导向）；端侧按 candidate/action/head 小规模并行（延迟导向），单条 rollout 沿时间串行。

数据复用：云端生成数据回流训练（跨任务复用）；端侧 history encoding 复用于多个 risk/cost 计算（`encode once → rollout many`）；模型更新跨端云传递。

量化敏感度：云端训练可混合精度；端侧安全相关输出（collision/occupancy）需校验量化误差，误差沿 rollout 累积。

瓶颈类型：云端是集群调度 + storage IO + 显存容量 + 跨卡通信 + 生成吞吐；端侧是 latency + 功耗 + state-cache 容量 + 确定性；协同回路是数据回流的 storage/上行/PCIe 带宽。

对硬件的核心需求：云端要训练/仿真吞吐 + HBM 容量 + 互连 + 数据管线；端侧要高效 state cache + 低延迟 rollout + 与感知/planning 共享中间表示 + 对外回流带宽——两套需求相反，详见 [Cloud Inference and Simulation Chip](../06-chip-workload-analysis/cloud-inference-and-simulation-chip.md) 与 [World Model Workload](../06-chip-workload-analysis/world-model-workload.md)。

## 参考来源

- NVIDIA, `Cosmos World Foundation Model Platform for Physical AI`, 2025 / arXiv:2501.03575。成熟度：2025 云端开源平台，数字孪生世界训练 policy，查证日期：2026-06-07。
- Waymo, `The Waymo World Model: A New Frontier For Autonomous Driving Simulation`, 2026-02。成熟度：产业前沿，云端闭环仿真，查证日期：2026-06-07。
- Caesar et al., `nuPlan: A closed-loop ML-based planning benchmark for autonomous vehicles`, 2021 / arXiv:2106.11810。成熟度：已落地，闭环评测基准。
- Dauner et al., `NAVSIM: Data-Driven Non-Reactive Autonomous Vehicle Simulation and Benchmarking`, NeurIPS 2024 / arXiv:2406.15349。成熟度：新兴评测基准。
- Assran et al., `V-JEPA 2: Self-Supervised Video Models Enable Understanding, Prediction and Planning`, 2025 / arXiv:2506.09985。成熟度：研究系统，端侧轻量 latent world model 代表，查证日期：2026-06-07。
</content>
