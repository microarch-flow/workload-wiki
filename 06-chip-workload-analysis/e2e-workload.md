# E2E Workload

上级：[Chip Workload Analysis](README.md)
相关：[Autonomous Driving Algorithms](../03-autonomous-driving-algorithms/README.md), [BEV Workload](bev-workload.md), [Occupancy Workload](occupancy-workload.md)

## 这页在回答什么问题

这页分析端到端自动驾驶的芯片压力。E2E 不是一个单一模型 workload，而是 sensor encoder、BEV/occupancy 表示、时序状态、多任务 head、planning decoder、safety shell 串成的**系统 workload**。它最容易被低估的不是某一段的算力，而是整条 sensor-to-control 路径的 worst-case latency，以及 CPU/NPU 之间的同步开销。

## 为什么 E2E 必须按 latency-critical path 拆，而不是按模型拆

一个 planning-oriented 端到端模型（如 UniAD）在算法视角下是"一个统一网络"，把检测、跟踪、运动预测、占用、规划用 query 串进一个 Transformer。但在 workload 视角下，这条链路是：

```text
sensor input → multi-modal encoder → BEV/query/occupancy 表示
  → detection/map/motion 辅助 head → planning/trajectory head
  → safety validation / controller
```

每一段的瓶颈类型都不同：encoder 是 compute-bound 的规则卷积/注意力；BEV 表示是 irregular-access-bound 的 view transform；时序融合是 stateful 的 cache 更新；planning head 是小张量、强依赖；safety/controller 往往跑在 CPU/安全岛上。把它们当一个 workload 平均，就会得到错误的芯片结论——平均 TOPS 可能很容易满足，但只要 view transform 或某个同步点造成 p99 尖峰，整车就无法部署。

## 系统瓶颈是同步，不是算力

E2E 自动驾驶真正难的地方是：感知 backbone 可以高效地跑在 NPU 上，但 BEV projection、tracking memory、planning head、collision check、规则约束、控制器之间存在大量 CPU/NPU 往返。每一次往返都是一个同步点，而 worst-case 同步开销决定 p99 latency。一颗芯片如果 NPU 算力很强但异构同步路径长，端到端延迟反而可能比算力较弱、但 pipeline 更紧凑、safety 近端执行的芯片更差。

因此 E2E workload 的设计目标不是平均吞吐最大化，而是 **bounded latency**：每一帧从 sensor 到 trajectory 的 worst-case path 必须稳定落在控制周期预算内（典型感知 10–30 Hz、规划/控制更高频）。

## 演进对 workload 的影响

E2E 的架构演进直接改变 workload 形态，这点对芯片选型很关键。UniAD（CVPR 2023，arXiv:2212.10156）确立了 query-based、planning-oriented 的密集 BEV+多任务范式，但计算重、延迟高。SparseDrive（arXiv:2405.19620）改用稀疏场景表示和并行运动规划，把推理速度做到比 UniAD 快约 5×——这说明同样是 E2E，稀疏化能显著改变 workload 的算力/延迟画像。DiffusionDrive（arXiv:2411.15139）在此基础上用截断扩散策略生成多模态轨迹，引入了"多步去噪"这一新的循环成本，但步数被截断以保实时。另一条线 EMMA（Waymo，arXiv:2410.23262）把端到端直接建在多模态大模型（Gemini）上、用文本表示统一感知与规划——这条线把 E2E 的 workload 拉向了 VLM 推理（见 [VLA Workload](vla-workload.md)），瓶颈从 BEV 不规则访问转移到大模型权重带宽与 token decode。架构探索时必须明确目标系统走的是哪条线，因为它们的瓶颈位置完全不同。

## 可建模参数

`camera count / resolution` 决定 encoder 算力与输入带宽；`history window` 决定时序 cache 容量；`BEV grid / occupancy grid` 决定特征/体素容量；`query count` 决定 decoder 算力与 attention 状态；`planning horizon` 决定轨迹输出与代价评估量；`candidate count` 决定多模态轨迹/rollout 的并行与容量；`sensor rate` 决定 p99 latency 预算。

## 硬件连接

RAM：BEV/occupancy/track/query 状态要常驻或高效换入换出，是 SRAM 容量规划的核心（见 RAM wiki）。DMA：sensor input、多 stage 中间特征、轨迹候选都需要 descriptor chaining 和 double buffering（见 DMA wiki）。NOC：encoder、BEV、多 head、planner 之间是 producer-consumer 流，需要 QoS 保证控制路径不被低优先级搬运阻塞（见 NOC wiki）。CIM：encoder、FFN、dense head 适合 CIM；planning validation、gather/scatter、动态 candidate selection 不适合。PCIE/host：CPU safety shell 与 NPU planner 的边界必须最小化，否则 p99 latency 不可控——这是 E2E 芯片架构最关键的 host-device boundary 决策。

## archax 建模

Resource：多 NPU tile、BEV SRAM、DRAM bandwidth、DMA channel，以及 CPU/DSP 的 safety 资源。Topology：`sensor ingress → NPU encoder → shared BEV memory → planner/safety` 的连接，特别是 safety island 与主计算的边界。Interaction：多 stage pipeline、状态 cache 更新、轨迹候选验证、CPU/NPU sync——其中同步点应被显式建模为有成本的交互，而不是零开销的边。Capability：Conv/GEMM/Attention、scatter-gather、stateful temporal cache、multi-head scheduling，以及最关键的 deterministic runtime。archax 探索 E2E 时，最有价值的扫描变量是 CPU/NPU 同步路径数和 BEV cache residency，而非 TOPS。

## 一句话理解

E2E 自动驾驶是一个由多段异质 stage 串成的系统 workload，成败由 worst-case sensor-to-control 延迟和 CPU/NPU 同步决定，而不是任何单段的峰值算力。

## Workload Characterization

- 计算密度：encoder 和 FFN/head compute-bound；BEV/occupancy/safety stage 偏 memory/latency-bound；整体取决于 stage 组合。
- 访存模式：同时存在连续 GEMM/Conv、BEV gather/scatter、时序状态 cache、轨迹 query 和 CPU/NPU 同步。
- 并行性：相机、BEV cell、query、task head、candidate 可并行；planning/safety 存在闭环依赖，不能完全流水。
- 数据复用：共享 BEV/query 特征是核心收益；历史状态和地图先验可跨帧复用。
- 量化敏感度：encoder/head 可量化；几何、轨迹回归、collision margin、红绿灯/小目标需谨慎。
- 瓶颈类型：第一瓶颈通常是 p99 latency 和 synchronization；局部叠加 BEV irregular-access 与 occupancy capacity。
- 对硬件的核心需求：端到端确定性 pipeline 调度、共享 state cache、低延迟 planner、safety 近端执行、可预测 runtime。

## 参考来源

- Hu et al., `Planning-oriented Autonomous Driving (UniAD)`, CVPR 2023 / arXiv:2212.10156。成熟度：已落地研究范式。
- Jiang et al., `VAD: Vectorized Scene Representation for Efficient Autonomous Driving`, ICCV 2023 / arXiv:2303.12077。成熟度：已落地，向量化高效范式。
- Sun et al., `SparseDrive: End-to-End Autonomous Driving via Sparse Scene Representation`, arXiv:2405.19620。成熟度：研究阶段，稀疏 E2E，推理较 UniAD 快约 5×。
- Liao et al., `DiffusionDrive: Truncated Diffusion Model for End-to-End Autonomous Driving`, CVPR 2025 / arXiv:2411.15139。成熟度：研究阶段，扩散式多模态轨迹。
- Hwang et al., `EMMA: End-to-End Multimodal Model for Autonomous Driving (Waymo)`, arXiv:2410.23262。成熟度：研究/产业原型，VLM-based E2E。
