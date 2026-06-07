# Data Closed Loop and Simulation

上级：[Autonomous Driving Algorithms](README.md)
相关：[Cloud Inference and Simulation Chip](../06-chip-workload-analysis/cloud-inference-and-simulation-chip.md), [World Model for Autonomous Driving](world-model-for-ad.md)

## 这页在回答什么问题

这页回答自动驾驶算法为什么离不开数据闭环与仿真。随着 E2E、VLM/VLA、World Model 进入系统，瓶颈不只是模型结构，而是如何持续发现长尾、重放失败、生成数据、训练模型并在闭环中评估。它的 workload 性格和前几篇的车端 batch=1 完全相反——这是一个云端吞吐主导的问题。

## 为什么它有效：直觉与类比

数据闭环的直觉是**把车队当成一支永不靠岸的捕捞船队，专捞"模型出错"的鱼**。每辆车都在路上撒网，捞的不是普通行驶数据（那些早就够了），而是稀有的长尾鱼——人类接管的瞬间、险些碰撞的 near miss、模型犹豫的路口、规则冲突的场景。这些"错题"被带回岸上的工厂，加工成可训练、可回放、可评估的数据资产，喂回模型，改进后的模型再放回海里继续捞更难的鱼。一个永不停的循环。

为什么这是自动驾驶的核心而非附属：因为驾驶的难度全在长尾。常规路况的数据多到没用，决定上限的是那 0.01% 的罕见场景，而你没法预先列举它们——只能靠车队规模在真实世界里持续打捞。所以现代自动驾驶的竞争，与其说是"谁的模型结构好"，不如说是**"谁的数据闭环转得快、长尾挖得准"**——护城河从模型架构迁移到了这座持续生产驾驶经验的工厂。

```text
fleet logs -> scenario mining / auto labeling / failure triage
   -> training data curation -> model training and validation
   -> simulation / closed-loop evaluation -> deployment and monitoring -> (回流)
```

## 为什么仿真必须是闭环：一个反直觉的陷阱

这里有个架构师必须理解的反直觉点：**open-loop 指标会骗人**。一条轨迹在 log 里看着和专家几乎重合（open-loop 误差很小），不代表它安全——因为真实驾驶是闭环的，你的动作会改变环境的反应。模型在 log 里"看着对"，可一旦它的轨迹让旁车被迫急刹、引发连锁反应，整个未来就偏离了 log，而 open-loop 评估根本看不到这层。这就是为什么 open-loop 轨迹误差无法充分反映真实安全，必须用闭环或 reactive 仿真才能暴露。

仿真因此分层，逼真度和价值递增：

| 层次 | 内容 | 适合用途 |
| --- | --- | --- |
| log replay | 重放真实传感器/中间表示 | 回归测试、离线评估 |
| non-reactive simulation | ego 变，环境基本不响应 | 快速 benchmark、planning 对比 |
| reactive simulation | 其他 agent 响应 ego | 闭环评估、交互场景 |
| generative simulation | world model/diffusion 生成未来 | 长尾扩增、what-if 测试 |

CARLA、nuPlan、NAVSIM 分别覆盖不同层级。World Model 的引入（见 [World Model for AD](world-model-for-ad.md)）让仿真从规则/资产驱动扩展到数据驱动生成——这是闭环测试从"能跑预设场景"走向"能造出没见过的长尾 counterfactual"的关键一跃。

## 一句话理解

数据闭环和仿真把自动驾驶从"训练一个模型"变成"持续生产、筛选、验证驾驶经验"的系统工程；它的护城河是长尾挖掘和闭环评估能力，而 open-loop 指标会骗人——只有 reactive/generative 闭环仿真才暴露真实安全。

## 演进与未来方向（判断）

以下为判断，锚定真实趋势。

主线判断：**护城河正从模型架构迁移到"数据闭环基础设施 + 生成式仿真"，而闭环评估本身正成为新瓶颈**。模型结构趋同后（大家都用 BEV/Occupancy/E2E/VLA），拉开差距的是谁能更快地从车队挖出长尾、更逼真地仿真出 counterfactual、更可信地度量闭环安全。生成式仿真（World Model 造长尾）是这条线上最热的方向，因为真实世界里罕见的场景，可以用可控生成批量造出来——GAIA-2、Waymo World Model、ReSim 这类工作正把"可靠世界仿真"当成核心目标。但有个硬约束：**生成质量不等于安全可验证**，造出逼真的事故视频容易，证明"在这个仿真里通过就等于真实安全"很难，可验证的 long-tail counterfactual generation 仍是前沿。

对架构师，这一篇的 workload 含义和 03 章其余各篇形成鲜明对照，也是 06 [Cloud Inference and Simulation Chip](../06-chip-workload-analysis/cloud-inference-and-simulation-chip.md) 的核心：**这是纯云端、吞吐主导的问题，不是车端 batch=1 的延迟问题**。它要的是大规模数据加载、批量推理、视频/点云解码、生成模型训练与推理、跨模型版本并发评测——关键资源是 storage bandwidth、cloud throughput、加速器显存和调度效率，数据湖 IO 常是真正的瓶颈而非算力。对 archax，这是一个和端侧推理正交的工作点族：端侧优化的是单流低延迟确定性，云端数据闭环优化的是多流高吞吐和显存容量。把这两类工作点分开建模，是因为为车端 NPU 做的架构选择（小 batch、确定性、低功耗）和为云端仿真集群做的选择（大 batch、高吞吐、大显存）几乎相反——这正是这份 wiki 区分端侧/云端芯片需求的依据。

## Workload Characterization

计算密度：云端回放、自动标注、仿真渲染、生成式扩增、多版本评测消耗巨大；车端仅日志采集和触发式上传。

访存模式：多传感器日志、标注、中间 feature、仿真状态、模型输出需高吞吐读写；数据湖 IO 常是瓶颈（容量与带宽，非算力）。

并行性：场景级、模型版本级、candidate policy 级高度并行；单个 reactive simulation 内部受时间步依赖限制。

数据复用：同一 log 可复用于感知训练、planner 评测、VLM 标注、World Model 学习、regression suite——一份数据多处摊销是闭环效率的关键。

量化敏感度：云端标注和仿真可混合精度；安全评估和 metric 计算需保持数值一致性（评测结果要可复现）。

瓶颈类型：主要是 cloud throughput、storage bandwidth、调度效率、数据治理；生成式仿真还受加速器显存制约——和车端的 latency-bound 完全不同。

对硬件的核心需求：高吞吐数据加载、批量推理、视频/点云解码、仿真渲染、生成模型训练、跨模型版本并发评测——这是云端吞吐机器，详见 [Cloud Inference and Simulation Chip](../06-chip-workload-analysis/cloud-inference-and-simulation-chip.md)。

## 参考来源

- Dosovitskiy et al., `CARLA: An Open Urban Driving Simulator`, CoRL 2017 / arXiv:1711.03938。成熟度：经典开源仿真平台。
- Caesar et al., `nuPlan: A closed-loop ML-based planning benchmark`, CVPR ADP3 2022 / arXiv:2106.11810。成熟度：常用规划闭环基准。
- Dauner et al., `NAVSIM: Data-Driven Non-Reactive Autonomous Vehicle Simulation and Benchmarking`, NeurIPS 2024 D&B / arXiv:2406.15349。成熟度：新兴 benchmark。
- `ReSim: Reliable World Simulation for Autonomous Driving`, arXiv:2506.09981。成熟度：2025 前沿研究，可靠世界仿真，查证日期：2026-06-07。
