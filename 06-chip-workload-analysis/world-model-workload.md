# World Model Workload

上级：[Chip Workload Analysis](README.md)
相关：[World Model and Generative Intelligence](../05-world-model-and-generative/README.md), [Diffusion for World Model](../05-world-model-and-generative/diffusion-for-world-model.md), [World Model Fundamentals](../05-world-model-and-generative/world-model-fundamentals.md)

## 这页在回答什么问题

这页分析 World Model 的芯片压力。它和前面所有 workload 的本质区别在于：成本不是一次 forward，而是一个**乘法**——`candidate count × horizon × per-step cost`，如果用 diffusion 或自回归视频生成，per-step 里还要再乘上去噪步数或 token 数。所以同样一个 World Model，用在端侧实时规划和用在云端数据生成，是两个相差几个数量级的 workload。

## 成本的乘法结构：World Model 与一次性推理的根本不同

感知模型一帧就是一次 forward。World Model 要"想象未来"：给定当前状态和一个候选动作序列，预测未来若干步的状态，并且通常要并行评估多个候选动作（planning 时挑最优、安全评估时覆盖多种情况）。于是总计算量是候选数乘以预测步长乘以单步模型成本。当模型是 latent 递推时单步很轻，这个乘法还能承受；当模型是 diffusion 视频生成时，单步本身就是几十次去噪迭代的大模型前向，乘法结果直接爆炸到只能放云端。

这个乘法结构决定了优化的着力点：**复用 condition encoding**。历史观测的编码、共享的条件，应该在所有 candidate 和 sample 之间只算一次，然后多个 rollout 复用——这是 World Model workload 最重要的省算手段，archax 里对应 Interaction 轴上的 `encode once → rollout many`。如果每个 candidate 都重新编码历史，乘法的底数会被无谓放大。

## 三类 World Model 的 workload 谱系

按表示形式，World Model 在轻重两端之间排成一个谱，对硬件的落点完全不同。

Latent World Model（如 Dreamer 系的 latent 递推）最轻：状态是低维 latent，单步递推是小型网络，适合端侧 candidate planning 的高频 rollout，瓶颈在 state cache 和候选并行度。Video World Model（如 GAIA-1 的离散 token 自回归、GAIA-2 的 latent diffusion、NVIDIA Cosmos）最重：要生成高分辨率多帧视频，单步是大生成模型，整体是 compute + capacity + IO 三重压力，基本只在云端做仿真和数据生成。BEV/Occupancy World Model（如 OccWorld 预测 4D 占用）介于两者之间，且更贴近自动驾驶的安全评估——它不追求像素逼真，而追求可控的结构化状态预测，这也呼应了"World Model ≠ 视频生成"的核心区分（见 [World Model Fundamentals](../05-world-model-and-generative/world-model-fundamentals.md)）。

需要点明成熟度：决策级、闭环可用的端侧 World Model 仍多处于研究阶段；已经落地的主要是云端的视频/场景生成与仿真（GAIA-2、Cosmos 这类作为数据引擎和仿真器）。把论文里的端侧实时 rollout 当成已量产能力，是这一方向最常见的误判。

## 可建模参数

`candidate count` 决定并发 rollout 数；`horizon` 决定 state cache 容量与循环次数；`latent/video/grid size` 决定容量与单步 decoder 算力；`denoise steps` 决定 diffusion 单步成本；`sample count` 决定多样未来的生成成本；`action dimension` 决定 dynamics 输入与候选 buffer；`closed-loop frequency` 决定端侧延迟预算。注意前四个参数是相乘关系，任何一个增大都按乘法放大总成本。

## 硬件连接

RAM：rollout state、future cache、video/occupancy latent 是容量压力的核心；云端视频生成对 HBM 容量和带宽要求极高（见 RAM wiki）。DMA：candidate state 与 condition cache 必须设计成可复用，避免每个 sample 重新搬运（见 DMA wiki）。NOC：多候选 rollout 并行时需要 state 分发和 cost 汇聚的路径（见 NOC wiki）。CIM：dense 去噪 block、GEMM/Conv 可用 CIM；rollout 循环、candidate selection、sparse query 不是 CIM 收益点。PCIE/host：云端仿真有大量生成数据回传（storage/NIC 压力），端侧实时 World Model 必须避免 host 参与每一步 rollout。

## archax 建模

Resource：rollout compute、state SRAM、DRAM/HBM bandwidth、DMA channel、candidate buffer。Topology：encoder/state cache 到 rollout/evaluator 的连接，以及云端训练/仿真集群的边界。Interaction：`encode once → rollout many → evaluate → select` 是核心交互，diffusion 还要加去噪循环——这两个循环是 archax 必须显式建模的成本乘子。Capability：stateful rollout、multi-candidate batching、diffusion 加速、early exit（对明显劣质候选提前终止）、结构化输出 query。archax 探索 World Model 时，最有价值的不是扫 TOPS，而是扫 candidate 并发度、condition cache 复用率和 horizon——它们决定那个乘法的量级。

## 一句话理解

World Model 的 workload 是一个 `候选 × 时域 × 单步（× 去噪步）` 的乘法：latent 递推轻到可上端侧、视频生成重到只能上云，而复用 condition encoding 是控制这个乘法的关键。

## Workload Characterization

- 计算密度：latent dynamics 中等；video/diffusion 高；occupancy decoder 容量压力大。
- 访存模式：history/state/candidate/future cache 持续读写；video/3D latent 容量大；结构化 query 不连续。
- 并行性：candidate、sample、scene 可高度并行；单条 rollout 的 horizon 和 denoise step 串行。
- 数据复用：history encoding 与 condition cache 在多 candidate/sample 间复用，是首要优化点。
- 量化敏感度：dense backbone 可低比特；长程一致性、collision/contact、occupancy 边界对误差累积敏感。
- 瓶颈类型：云端是 compute + capacity + IO；端侧是 latency + state capacity + candidate 数。
- 对硬件的核心需求：多候选并发、长时序 state cache、生成模型加速、condition cache 复用、清晰的端云分工。

## 参考来源

- Hafner et al., `Mastering Diverse Domains through World Models (DreamerV3)`, arXiv:2301.04104。成熟度：已落地研究，latent world model 代表。
- Hu et al., `GAIA-1: A Generative World Model for Autonomous Driving`, arXiv:2309.17080。成熟度：研究阶段，离散 token 自回归视频世界模型。
- Russell et al., `GAIA-2: A Controllable Multi-View Generative World Model for Autonomous Driving (Wayve)`, arXiv:2503.20523。成熟度：2025 产业研究，latent diffusion 多视角世界模型。
- Gao et al., `Vista: A Generalizable Driving World Model with High Fidelity and Versatile Controllability`, NeurIPS 2024 / arXiv:2405.17398。成熟度：研究阶段。
- NVIDIA, `Cosmos World Foundation Model Platform for Physical AI`, arXiv:2501.03575。成熟度：2025 产业平台，云端仿真/数据生成。
