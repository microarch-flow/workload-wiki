# Action Tokenizer

上级：[Robotics and VLA](README.md)
相关：[VLA Fundamentals](vla-fundamentals.md), [VLA Workload](../06-chip-workload-analysis/vla-workload.md)

## 这页在回答什么问题

这页回答 Action Tokenizer 为什么是 VLA 的核心问题。语言和图像天然可以 token 化，但机器人动作是连续、高频、带物理约束的信号；如果动作表示不好，大模型即使理解任务，也难以稳定控制机器人。

## 动作表示的几种方式

| 表示 | 例子 | 优点 | 问题 |
| --- | --- | --- | --- |
| continuous regression | 直接输出 joint delta / pose delta | 延迟低，控制自然 | 难与 LLM token 空间统一 |
| discrete bins | 把每个维度量化成 bins | 可接入 autoregressive LM | token 长、精度和抖动难平衡 |
| action chunk | 一次输出未来多步动作 | 降低推理频率 | 闭环响应变慢 |
| latent action token | VQ-VAE / FAST 类 tokenizer | 压缩动作序列，适合大模型 | 需要解码器恢复连续控制 |
| flow / diffusion action head | 条件生成连续 action | 多模态动作表达强 | 推理步数和实时性是压力 |

## 为什么 tokenizer 会影响 workload

如果把 7-DoF 机械臂每个维度都离散化，再按时间步 autoregressive 输出，token 数会迅速增加。token 越多，decoder latency、KV cache 和串行依赖越明显。好的 action tokenizer 要在三件事之间折中：动作精度、token 长度、解码稳定性。

FAST 这类工作试图用更高效的动作 tokenization 压缩连续 action trajectory，使 VLA 可以更像语言模型一样预测动作，同时减少 token 数和推理开销。另一条路线是 flow/diffusion action head，让模型生成连续 action chunk，减少离散 token 的量化损失。

常见误解：action token 越细越好。实际上，过细会增加 token 长度和延迟，过粗会导致控制抖动或无法完成精细操作；tokenizer 是控制质量和硬件成本之间的接口。

## 成熟度判断

- 成熟：continuous action regression、action chunking、Diffusion Policy。
- 发展中：VLA 中的离散 action token 和 latent action tokenizer。
- 前沿：通用 action tokenizer、flow matching action head、跨 embodiment action space 对齐。

## 一句话理解

Action Tokenizer 把连续机器人控制变成模型可生成的符号或 latent；它决定 VLA 是低延迟控制问题，还是长 token 序列生成问题。

## Workload Characterization

- 计算密度：tokenizer/decoder 本身通常不重；真正影响来自 action token 数增加后导致的 Transformer decode 开销。
- 访存模式：autoregressive action token 依赖 KV cache；action chunk 和 latent decoder 需要短窗口动作缓存。
- 并行性：连续 regression 可一次并行输出；autoregressive token 串行；diffusion/flow 可在 sample 或 denoise step 上并行但有迭代成本。
- 数据复用：同一 observation encoding 可复用于多个 action candidate；action chunk 可复用一次视觉编码覆盖多个控制周期。
- 量化敏感度：token embedding 可量化；末端连续动作恢复、gripper 开闭、接触阶段动作边界对精度敏感。
- 瓶颈类型：高频控制场景中，token decode latency 比单次 FLOPs 更关键；训练侧还受动作数据标准化和跨机器人对齐影响。
- 对硬件的核心需求：低延迟小 batch decode、KV cache 高效访问、短序列实时调度、连续动作后处理、与低层控制器稳定交互。

## 参考来源

- Chi et al., `Diffusion Policy: Visuomotor Policy Learning via Action Diffusion`, RSS 2023 / arXiv:2303.04137，https://arxiv.org/abs/2303.04137，成熟度：成熟研究路线，查证日期：2026-05-29。
- Zitkovich et al., `RT-2`, CoRL 2023 / arXiv:2307.15818，动作作为 text-like tokens，https://arxiv.org/abs/2307.15818，成熟度：VLA action token 早期代表，查证日期：2026-05-29。
- `FAST: Efficient Action Tokenization for Vision-Language-Action Models`, arXiv:2501.09747，https://arxiv.org/abs/2501.09747，成熟度：2025 前沿研究，查证日期：2026-05-29。
- `SmolVLA: Efficient Vision-Language-Action Model trained on Lerobot Community Data`, arXiv:2506.01844，https://arxiv.org/abs/2506.01844，成熟度：2025 高效 VLA 研究，查证日期：2026-05-29。
