# VLA Workload

上级：[Chip Workload Analysis](README.md)
相关：[Robotics and VLA](../04-robotics-and-vla/README.md), [Transformer Workload](transformer-workload.md)

## 这页在回答什么问题

这页分析 Vision-Language-Action workload。VLA 的芯片压力来自 VLM backbone、visual token、language token、robot state、KV cache、action tokenizer 和低延迟闭环控制的组合。它不是普通 LLM 推理，也不是轻量机器人 policy。

## Stage 拆解

| Stage | 输入输出 | 主导成本 |
| --- | --- | --- |
| visual encoder | RGB/wrist camera -> visual tokens | Conv/ViT compute、activation |
| language/context encoder | instruction/history -> text tokens | prefill compute |
| multimodal fusion | visual + language + state | attention/KV cache |
| action decode | action token / action chunk | autoregressive latency 或 continuous head |
| safety/control bridge | action -> low-level controller | p99 latency、CPU/NPU sync |

RT-2、OpenVLA、GR00T、π0/π0.5、FAST、SmolVLA 等路线说明 VLA 正在从大 VLM 直接输出动作，演进到更高效 action tokenizer、action chunk、flow/diffusion action head 和跨 embodiment policy。

## 关键参数

| 参数 | 放大什么 |
| --- | --- |
| `visual token count` | attention compute、KV cache |
| `LLM/VLM parameter size` | 权重带宽、memory capacity |
| `action token length` | decode latency |
| `action chunk length` | 控制频率与闭环响应 |
| `control frequency` | 端侧实时预算 |
| `camera count` | visual encoder 与 token 数 |
| `robot state dimension` | fusion 和 action head |

## 硬件连接

- RAM：VLM 权重、KV cache、visual tokens 和 action history 共同占容量。
- DMA：camera input、token buffer、KV cache update 需要低延迟搬运。
- NOC：视觉 encoder、LLM block、action head 之间共享 token/state。
- CIM：GEMM/FFN/QKV projection 可用 CIM；action decode 的状态循环和安全控制不是 CIM 主收益点。
- PCIE/host：低层控制器、传感器和安全监控若跨 host，会直接影响控制稳定性。

## archax 建模

- Resource：VLM TOPS、SRAM/KV cache、DRAM bandwidth、camera ingress、controller interface。
- Topology：visual encoder -> token fusion -> decode loop -> controller 的低延迟路径。
- Interaction：prefill、decode、action chunk rollout、robot state update、safety monitor sync。
- Capability：GEMM/Attention、KV cache quantization、action tokenizer、stateful decode、mixed precision、bounded latency scheduling。

## Workload Characterization

- 计算密度：prefill/visual encoding compute-bound；action decode 常 latency/KV-bandwidth-bound。
- 访存模式：权重和 visual tokens 连续；KV cache 和 action history 是状态访问；多相机增加输入带宽。
- 并行性：视觉编码、head、候选动作可并行；autoregressive action token 串行。
- 数据复用：instruction/context 可在短任务中复用；视觉 token 更新频繁；action chunk 可摊薄编码成本。
- 量化敏感度：VLM 权重可 4/8-bit；action head、gripper/contact、state embedding 需要保守验证。
- 瓶颈类型：端侧第一瓶颈是 memory capacity + per-action latency；训练侧瓶颈是多模态数据吞吐和模型规模。
- 硬件核心需求：低 batch VLM、KV cache 管理、action decode 低延迟、多传感器输入、控制器近端同步。

## 参考来源

- Zitkovich et al., `RT-2`, CoRL 2023 / arXiv:2307.15818，成熟度：VLA 范式代表，查证日期：2026-05-29。
- Kim et al., `OpenVLA`, arXiv:2406.09246，成熟度：开源 VLA baseline，查证日期：2026-05-29。
- Black et al., `π0: A Vision-Language-Action Flow Model for General Robot Control`, arXiv:2410.24164，成熟度：2024 前沿研究，查证日期：2026-05-29。
- `π0.5: a Vision-Language-Action Model with Open-World Generalization`, arXiv:2504.16054，成熟度：2025 前沿研究，查证日期：2026-05-29。
- NVIDIA, `GR00T N1`, arXiv:2503.14734，成熟度：2025 产业研究原型，查证日期：2026-05-29。
- `FAST: Efficient Action Tokenization for Vision-Language-Action Models`, arXiv:2501.09747，成熟度：2025 前沿研究，查证日期：2026-05-29。
- `SmolVLA: Efficient Vision-Language-Action Model trained on Lerobot Community Data`, arXiv:2506.01844，成熟度：2025 高效 VLA 研究，查证日期：2026-05-29。
- `/mnt/e/workload-wiki-old/06_芯片架构Workload分析/VLA_Workload.md`。
