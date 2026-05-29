# World Model Workload

上级：[Chip Workload Analysis](README.md)
相关：[World Model and Generative Intelligence](../05-world-model-and-generative/README.md), [Diffusion for World Model](../05-world-model-and-generative/diffusion-for-world-model.md)

## 这页在回答什么问题

这页分析 World Model workload。World Model 的成本不是单次 forward，而是 `candidate count x horizon x model step cost`，如果使用 diffusion 或 autoregressive video，还要乘上 denoise step 或 token step。

## Stage 拆解

| Stage | 输入输出 | 主导成本 |
| --- | --- | --- |
| state encoder | history -> latent/BEV/video state | encoder compute |
| dynamics model | state + action -> future state | recurrent/Transformer rollout |
| generative decoder | latent -> video/occupancy | diffusion/video/3D decode |
| candidate evaluator | future -> risk/cost/value | 多候选并发 |
| data/simulation loop | generated scene -> training/eval | 云端吞吐 |

Latent World Model 较轻，适合 candidate planning；Video World Model 最重，适合云端仿真和数据生成；BEV/Occupancy World Model 介于两者之间，更贴近自动驾驶安全评估。

## 关键参数

| 参数 | 放大什么 |
| --- | --- |
| `candidate count` | 并发 rollout 数 |
| `horizon` | state cache、循环次数 |
| `latent/video/grid size` | capacity、decoder compute |
| `denoise steps` | diffusion 推理成本 |
| `sample count` | 多样未来生成成本 |
| `action dimension` | dynamics input 和 candidate buffer |
| `closed-loop frequency` | 端侧 latency 预算 |

## 硬件连接

- RAM：rollout state、future cache、video/occupancy latent 是容量压力核心。
- DMA：candidate state 与 condition cache 需要复用，避免每个 sample 重搬。
- NOC：多候选 rollout 并行时，需要 state 分发和 cost 汇聚。
- CIM：dense denoise block、GEMM/Conv 可用；rollout loop、candidate selection、sparse query 不是 CIM 主收益点。
- PCIE/host：云端仿真会有大量数据回传；端侧实时 world model 应避免 host 参与每步 rollout。

## archax 建模

- Resource：rollout compute、state SRAM、DRAM/HBM bandwidth、DMA channel、candidate buffer。
- Topology：encoder state cache 到 rollout/evaluator 的连接，云端训练/仿真集群边界。
- Interaction：`encode once -> rollout many -> evaluate -> select`，以及 diffusion denoise loop。
- Capability：stateful rollout、multi-candidate batching、diffusion acceleration、early exit、structured output query。

## Workload Characterization

- 计算密度：latent dynamics 中等；video/diffusion 高；occupancy decoder 容量压力大。
- 访存模式：history/state/candidate/future cache 持续读写；video/3D latent 容量大；sparse query 不连续。
- 并行性：candidate、sample、scene 可并行；单 rollout horizon 和 denoise step 串行。
- 数据复用：history encoding 和 condition cache 可在多个 candidate/sample 间复用，是关键优化点。
- 量化敏感度：dense backbone 可低比特；长期一致性、collision/contact、occupancy boundary 对误差敏感。
- 瓶颈类型：云端是 compute + capacity + IO；端侧是 latency + state capacity + candidate 数。
- 硬件核心需求：多候选并发、长时序 state cache、生成模型加速、condition cache 复用、端云分工。

## 参考来源

- Hafner et al., `DreamerV3`, arXiv:2301.04104。
- Hu et al., `GAIA-1`, arXiv:2309.17080。
- NVIDIA, `Cosmos World Foundation Models`, 2025，成熟度：产业平台，查证日期：2026-05-29。
- `DriveWAM`, arXiv:2605.28544，成熟度：2026 前沿研究，查证日期：2026-05-29。
- `/mnt/e/workload-wiki-old/06_芯片架构Workload分析/World_Model_Workload.md`。
