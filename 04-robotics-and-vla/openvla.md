# OpenVLA

上级：[Robotics and VLA](README.md)
相关：[VLA Fundamentals](vla-fundamentals.md), [Action Tokenizer](action-tokenizer.md), [VLA Workload](../06-chip-workload-analysis/vla-workload.md)

## 这页在回答什么问题

这页拆解 OpenVLA 的定位和 workload。OpenVLA 的重要性在于它把 VLA 做成开源可复现实验平台，使研究者可以在开源 VLM、机器人 demonstration 和 action tokenizer 之间做系统性权衡。

## 模型结构

OpenVLA 采用 pretrained VLM 作为基础，再在机器人数据上训练 action output。典型数据流是：

```text
RGB observation + language instruction
   ->
vision encoder + language backbone
   ->
multimodal hidden states
   ->
action token prediction
   ->
robot control command
```

OpenVLA 原始模型规模为 7B 级别，适合研究和离线部署验证，但直接上低功耗端侧机器人并不轻。后续社区工作开始围绕参数压缩、LoRA fine-tuning、FAST tokenizer、SmolVLA 等方向降低部署成本。

## 为什么它重要

OpenVLA 让 VLA 从少数闭源系统变成可被复现、压缩和微调的开源路线。对架构师来说，它提供了一个典型 workload 样本：大 VLM backbone、视觉 token 输入、动作 token decode、batch=1 低延迟、低功耗部署。

常见误解：OpenVLA 等于可直接通用部署的机器人“大脑”。实际上，它更适合作为开源 VLA baseline；真实部署还需要机器人本体适配、相机标定、控制安全、任务数据 fine-tuning 和低层控制器。

## 部署压力

OpenVLA 的推理压力来自三个方面。第一是模型参数和 KV cache，7B 级模型对端侧显存或片上/片外带宽要求高。第二是视觉输入，机器人通常至少有第三视角相机，很多场景还需要 wrist camera 或 depth。第三是动作输出，若采用 autoregressive token，会让控制频率受到 token latency 限制。

## 一句话理解

OpenVLA 是机器人 VLA 的开源锚点：它证明开源 VLM 可以接机器人动作，但也暴露了大模型控制在端侧部署时的 latency、memory 和动作表示压力。

## Workload Characterization

- 计算密度：VLM backbone 占主导，主要是大矩阵乘和 attention；action token head 相对轻。
- 访存模式：权重加载、KV cache、视觉 token 和动作 token 序列是主要访存对象；batch=1 时权重复用差。
- 并行性：视觉编码可并行；LLM decode 串行；多个 action candidate 或 beam 可批并行但增加延迟。
- 数据复用：instruction 和部分 context 可在短任务内复用；视觉 token 每帧更新，复用窗口短。
- 量化敏感度：7B backbone 可用 4/8-bit 量化探索；action token logits、末端动作恢复和安全边界需要任务级验证。
- 瓶颈类型：端侧瓶颈通常是 memory capacity + token latency；云端 fine-tuning 瓶颈是多模态 batch 和机器人数据加载。
- 对硬件的核心需求：LLM/VLM 低 batch 推理、KV cache 带宽、视觉编码、低延迟 action decode、量化后稳定性。

## 参考来源

- Kim et al., `OpenVLA: An Open-Source Vision-Language-Action Model`, arXiv:2406.09246，https://arxiv.org/abs/2406.09246，成熟度：开源研究 baseline，查证日期：2026-05-29。
- OpenVLA GitHub, https://github.com/openvla/openvla，成熟度：开源实现，查证日期：2026-05-29。
- `FAST: Efficient Action Tokenization for Vision-Language-Action Models`, arXiv:2501.09747，https://arxiv.org/abs/2501.09747，成熟度：2025 前沿 tokenizer，查证日期：2026-05-29。
- `SmolVLA: Efficient Vision-Language-Action Model trained on Lerobot Community Data`, arXiv:2506.01844，https://arxiv.org/abs/2506.01844，成熟度：2025 高效 VLA 研究，查证日期：2026-05-29。
