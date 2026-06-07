# OpenVLA

上级：[Robotics and VLA](README.md)
相关：[VLA Fundamentals](vla-fundamentals.md), [Action Tokenizer](action-tokenizer.md), [RT Series](rt-series.md), [VLA Workload](../06-chip-workload-analysis/vla-workload.md), [Attention and Transformer](../01-foundation-model-components/attention-and-transformer.md)

## 这页在回答什么问题

这页把 OpenVLA 当一个可计量的 workload 样本拆开。它的意义不在性能纪录，而在它是第一个把 VLA 做成完全开源、参数和数据流都公开可复现的 7B 模型——于是它成了"机器人 VLA 的端侧部署到底卡在哪"这个问题最干净的研究对象。这页给出它的真实结构数字，端侧资源分解的完整版在 [VLA Workload](../06-chip-workload-analysis/vla-workload.md)。

## 为什么它有效：直觉与类比

OpenVLA 的设计直觉是**用两副眼睛 + 一个会说话的大脑，最省事地接上动作**。两副眼睛是 DINOv2 和 SigLIP 两个视觉 encoder 并联——一个像测绘员盯空间结构和精确位置（DINOv2 的自监督特征对几何/空间保真），一个像鉴赏家盯语义类别（SigLIP 的图文对比特征对"这是什么"敏感）。对应到机制：两路视觉特征拼接后投进语言模型的 token 空间，让 backbone 同时拿到"东西在哪"和"东西是什么"——单一视觉 encoder 往往在这两件事上顾此失彼，并联是用一点额外编码成本换更全的视觉表示，这对要精确够到物体的操作任务是直接收益。

大脑是 Llama-2 7B，接动作的方式最省事：**把动作塞进它本就有的词表**。OpenVLA 把 7 个自由度（x, y, z, roll, pitch, yaw, gripper）各量化成 256 个 bin，再用这 256 个动作 token 覆写 Llama-2 词表里最少用的 256 个 token。对应到机制：这样动作生成完全复用语言模型的 next-token prediction，不加新参数、不改 backbone 结构，训练就是标准交叉熵——这是"零改架构接动作"的极致，也是它好复现的原因。单步动作就是 7 个 token，autoregressive 左到右生成。

省事的代价是它把所有 VLA 的端侧痛点都原样继承下来，而且因为开源，这些痛点第一次被完整暴露出来给所有人研究。对应到机制：7B backbone + batch=1 decode 意味着每出一个动作 token 都要把约 7B 权重从 HBM 过一遍（memory-bandwidth-bound），7 个 token 串行；视觉端两个 encoder 加多相机让 prefill 的 token 数不小；动作精度被 256 bin 的量化粒度卡住。OpenVLA 之所以重要，正是因为它把"大模型控制在端侧到底贵在哪"这件事变成了一个谁都能跑、能压、能改的公开 benchmark——后续 LoRA 微调、4-bit 量化、FAST tokenizer、SmolVLA 全是拿它当靶子做的优化。

## 真实结构与数字锚点

OpenVLA 的数据流：RGB observation + language instruction → DINOv2 + SigLIP 双视觉 encoder（基于 Prismatic prism-dinosiglip-224px VLM）→ 投影到 Llama-2 7B token 空间 → 多模态 hidden states → 7 个动作 token（各 256 bin）→ 反量化成 7-DoF end-effector delta → 机器人控制。

几个该记住的量。模型 7B 量级，训练数据是 Open X-Embodiment 里约 97 万条轨迹的混合。动作离散化的一个细节很有 workload 含义：每个标量动作映射到 1-99 百分位之间均匀切的 256 个 bin（裁掉离群值），这把动作分辨率固定在了 1/256——精细操作要么不够，要么得靠 action chunk 或换 FAST/flow head 补。224px 单相机经视觉 encoder 约产生 256 个 visual token，多相机（第三视角 + wrist camera）线性叠加，是 prefill token 数的主要来源。

OpenVLA 原始 7B 适合研究和离线验证，直接上低功耗端侧机器人并不轻——这正是它催生一整片压缩工作的原因：LoRA 微调降训练成本、4/8-bit 量化降显存、FAST tokenizer 压动作 token 数、SmolVLA 整体下沉到消费级硬件。

## 它暴露的三处部署压力

OpenVLA 的推理压力清楚地分三块，每块都对应 06 的一类资源。第一是**模型参数和 KV cache**：7B 权重 + decode 时的 KV cache 对端侧显存/带宽要求高，batch=1 下权重无法在 batch 维摊销，每 token 都是一次全权重 HBM 往返。第二是**视觉输入**：机器人至少一个第三视角相机，很多场景还要 wrist camera 或 depth，多路视觉 token 抬高 prefill 成本。第三是**动作输出**：autoregressive 的 7 token 串行 decode 让控制频率受 token latency 直接限制——这也是 FAST、flow head 想替换掉的环节。

## 一句话理解

OpenVLA 是机器人 VLA 的开源锚点：DINOv2+SigLIP 双视觉 encoder + Llama-2 7B backbone + 7 维各 256 bin 的 text-token 动作，证明开源 VLM 能接机器人动作；但它把 7B batch=1 decode 的 memory-bandwidth 瓶颈、多相机 prefill、量化敏感的动作边界这三处端侧压力完整暴露，成了所有压缩优化的公共靶子。

## 演进与未来方向（判断）

以下为判断，锚定 2025-2026 联网核实的真实工作，查证日期：2026-06-07。

OpenVLA 的历史角色是"把 VLA 从闭源变成可解剖的公共平台"，它本身那种 7B + 逐维 binning + 串行 text-token decode 的形态，在 2025-2026 已被它自己催生的优化群超越。三条线都拿它当起点：动作表示上，π0-FAST 用 DCT+BPE 把它那 7 token/步的串行 decode 压短，或像 π0 直接换 flow-matching action expert；模型尺寸上，SmolVLA 把 7B 下沉到能在消费级甚至端侧硬件跑；机制研究上，社区拿 OpenVLA 做了大量可解释性和鲁棒性工作（哪些视觉特征真正驱动动作、视觉 token 能剪多少），这些只在开源模型上做得了。

我的判断是，**OpenVLA 作为"研究 baseline 和靶子"会长期存活，但作为"部署形态"已被取代**：它最大的价值是让"VLA 端侧到底卡在哪"变成可测量、可对比的公共问题，而答案——batch=1 decode 的 memory-bandwidth、多相机 prefill、动作量化敏感——一旦量化清楚，优化方向（压 token、压尺寸、换 action head）就跟着确定了。具体到 visual token：研究已表明 OpenVLA 的视觉 token 有相当冗余可剪，这意味着 prefill 阶段的 token 预算是个可压缩量，而非固定成本。

对架构师，OpenVLA 的价值正是它的可计量性：它给出了一个干净的 7B-batch=1-端侧 VLA 工作点，参数、token 数、动作维度全公开，可以直接喂进 archax 当一个标定样本。它清楚地说明，机器人 VLA 端侧 workload 的主成本不在算力（FLOPs 不算夸张）而在 memory-bandwidth 和 KV cache——这把架构判断的重心从"堆算力"挪到"喂数据"，正是 06 [VLA Workload](../06-chip-workload-analysis/vla-workload.md) 要展开的核心。

## Workload Characterization

以 OpenVLA 7B、DINOv2+SigLIP 双 encoder、224px、batch=1 端侧为基准。

计算密度：Llama-2 7B backbone 占绝对主导，主要是大 GEMM 和 attention，单 token decode 约 2×7B ≈ 14 GFLOP；双视觉 encoder 在 prefill 一次性编码（DINOv2 + SigLIP 并联，编码成本约两个 ViT）；7 个动作 token 的 logits 投影极轻。

访存模式：batch=1 decode 下权重无 batch 复用，每 token 一次约 7B 参数（INT8 约 7GB、4-bit 约 3.5GB）的 HBM 往返，是 memory-bandwidth-bound 主因；KV cache、约 256 个/相机的 visual token、7 个动作 token 序列共占容量；多相机线性叠加 prefill token。

并行性：双视觉 encoder 和多相机可并行；Llama-2 decode 严格串行（7 token 因果依赖）；多 action candidate 采样可批并行但增延迟；prefill 的 visual+language token 可并行。

数据复用：instruction 和短任务 context 的 KV 可跨控制周期复用；视觉 token 每帧更新、复用窗口短；同一份 hidden state 可服务动作、成功判断、解释多 head。

量化敏感度：7B backbone 可 4/8-bit 量化探索（社区已验证 4-bit 可用）；但动作 token 的 logits、256-bin 反量化恢复、gripper 与接触边界对精度敏感，需任务级而非 backbone 级的量化验证。

瓶颈类型：端侧瓶颈是 memory-capacity（装下 7B + KV cache）+ token-latency（7 token 串行决定控制频率）；云端 fine-tuning 瓶颈是多模态 batch 和机器人数据加载；visual token 经研究证明有可剪冗余，prefill 成本部分可压。

对硬件的核心需求：7B LLM/VLM 的低 batch 推理通路、KV cache 带宽与容量、双视觉 encoder 的并行编码、7 动作 token 的低延迟串行 decode、4/8-bit 量化后的数值稳定性——详见 [VLA Workload](../06-chip-workload-analysis/vla-workload.md)。

## 参考来源

- Kim et al., `OpenVLA: An Open-Source Vision-Language-Action Model`, arXiv:2406.09246。成熟度：开源研究 baseline（Llama-2 7B + DINOv2/SigLIP，7-DoF 各 256 bin，约 97 万 OXE 轨迹训练）。查证日期：2026-06-07。
- OpenVLA 项目仓库，https://github.com/openvla/openvla 与 huggingface.co/openvla/openvla-7b。成熟度：开源实现与权重。查证日期：2026-06-07。
- Pertsch et al., `FAST: Efficient Action Tokenization for Vision-Language-Action Models`, arXiv:2501.09747。成熟度：2025 研究阶段，压缩 OpenVLA 类动作 token 数的代表方法。查证日期：2026-06-07。
- `SmolVLA: A Vision-Language-Action Model for Affordable and Efficient Robotics`, arXiv:2506.01844。成熟度：2025 研究阶段，把 VLA 下沉到消费级硬件。查证日期：2026-06-07。
