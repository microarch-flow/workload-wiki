# E2E Workload

上级：[Chip Workload Analysis](README.md)
相关：[Autonomous Driving Algorithms](../03-autonomous-driving-algorithms/README.md), [BEV Workload](bev-workload.md), [Occupancy Workload](occupancy-workload.md)

## 这页在回答什么问题

这页分析端到端自动驾驶 workload。E2E 不是一个单一模型 workload，而是 sensor encoder、BEV/occupancy、temporal state、multi-task heads、planning decoder、safety shell 组成的系统 workload。架构建模必须按 latency-critical path 拆 stage。

## Stage 拆解

```text
sensor input
  -> multi-modal encoder
  -> BEV / query / occupancy representation
  -> detection / map / motion auxiliary heads
  -> planning / trajectory head
  -> safety validation / controller
```

planning-oriented E2E 的关键是共享表征服务最终轨迹。对芯片来说，这意味着 encoder 和 BEV feature 可以复用，但 multi-head、temporal cache、planner 和 safety stage 会引入同步点。

## 关键参数

| 参数 | 放大什么 |
| --- | --- |
| `camera count / resolution` | encoder compute、input bandwidth |
| `history window` | temporal cache 容量和状态更新 |
| `BEV grid / occupancy grid` | feature/voxel capacity |
| `query count` | decoder compute、attention state |
| `planning horizon` | trajectory output、cost evaluation |
| `candidate count` | rollout/evaluation 并行和容量 |
| `sensor rate` | p99 latency 预算 |

## 系统瓶颈

E2E 自动驾驶最容易被低估的是同步。感知 backbone 可以高效跑在 NPU 上，但 BEV projection、tracking memory、planning head、collision check、规则约束和控制器之间的 CPU/NPU 往返会决定 p99 latency。

因此，E2E workload 的目标不是平均 TOPS 最大化，而是 bounded latency：每一帧从 sensor 到 trajectory 的 worst-case path 必须稳定。

## 硬件连接

- RAM：BEV/occupancy/track/query state 要常驻或高效换入换出。
- DMA：sensor input、multi-stage feature、trajectory candidate 需要 descriptor chaining 和 double buffering。
- NOC：encoder、BEV、multi-head、planner 之间有 producer-consumer 流。
- CIM：encoder、FFN、dense heads 适合；planning validation、gather/scatter、动态 candidate selection 不适合作为主收益点。
- PCIE/host：CPU safety shell 与 NPU planner 的边界必须最小化，否则 p99 latency 不可控。

## archax 建模

- Resource：多 NPU tile、BEV SRAM、DRAM bandwidth、DMA channel、CPU/DSP safety resource。
- Topology：sensor ingress -> NPU encoder -> shared BEV memory -> planner/safety。
- Interaction：multi-stage pipeline、state cache update、trajectory candidate validation、CPU/NPU sync。
- Capability：Conv/GEMM/Attention、scatter-gather、stateful temporal cache、multi-head scheduling、deterministic runtime。

## Workload Characterization

- 计算密度：encoder 和 FFN/head compute-bound；BEV/occupancy/safety stage 更偏 memory/latency-bound。
- 访存模式：包含连续 GEMM/Conv、BEV gather/scatter、state cache、trajectory query 和 CPU/NPU 同步。
- 并行性：camera、BEV cell、query、task head、candidate 可并行；planning/safety 有闭环依赖。
- 数据复用：共享 BEV/query feature 是核心；history state 和 map prior 可跨帧复用。
- 量化敏感度：encoder/head 可量化；geometry、trajectory regression、collision margin、traffic light/small object 需谨慎。
- 瓶颈类型：第一瓶颈常是 p99 latency 和 synchronization；局部瓶颈包括 BEV irregular access 与 occupancy capacity。
- 硬件核心需求：端到端 pipeline 调度、共享 state cache、低延迟 planner、安全 shell 近端执行、可预测 runtime。

## 参考来源

- Hu et al., `Planning-oriented Autonomous Driving`, CVPR 2023 / arXiv:2212.10156。
- Jiang et al., `VAD: Vectorized Scene Representation for Efficient Autonomous Driving`, ICCV 2023 / arXiv:2303.12077。
- Caesar et al., `nuPlan`, arXiv:2106.11810。
- `/mnt/e/workload-wiki-old/06_芯片架构Workload分析/E2E_Workload.md`。
