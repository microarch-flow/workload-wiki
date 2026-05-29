# VLA Fundamentals

上级：[Robotics and VLA](README.md)
相关：[VLA Workload](../06-chip-workload-analysis/vla-workload.md), [Transformer](../01-foundation-model-components/attention-and-transformer.md)

## 这页在回答什么问题

这页回答 VLA 的基本范式：为什么机器人模型要把 vision、language 和 action 放在一起建模，以及这会怎样改变计算结构。VLA 不是普通 VLM 后面接一个控制器，而是让动作成为模型输出空间的一部分。

## 基本结构

```text
camera / wrist camera / robot state / language instruction
   ->
vision encoder + language model + state embedding
   ->
shared multimodal tokens
   ->
action decoder
   ->
joint delta / gripper / end-effector pose / action chunk
```

机器人任务中，language 给出任务目标，vision 给出当前环境，robot state 给出本体姿态，action decoder 把共享表示转成连续控制或离散 action token。与纯 VLM 相比，VLA 多了闭环控制约束：输出必须在几十毫秒到数百毫秒内被机器人执行，并且连续动作误差会立即改变下一帧 observation。

## 为什么需要 VLA

传统机器人 pipeline 把感知、规划、控制拆开，适合固定任务，但很难覆盖开放指令和多物体交互。VLA 的价值在于把 web-scale 视觉语言知识、机器人 demonstration 和动作策略放进同一训练框架，使模型有机会泛化到新物体、新指令和新场景。

常见误解：VLA 的重点是“能听懂自然语言”。实际上，语言只是任务条件的一种形式；更关键的是把动作与视觉语义对齐，并让 action head 继承大模型的表示能力。

## 当前演进

RT-2 把 robot action 表示为 text-like tokens，使 VLM 可以直接输出动作。OpenVLA 进一步把开源 VLM 预训练与机器人 demonstration 结合。2025 年前后，π0/π0.5、GR00T N1、FAST、SmolVLA 等工作把重点推向更高频控制、更高效 action representation、跨 embodiment 泛化和可部署模型尺寸。

成熟度上，VLA 已经是机器人学习的重要研究范式；但跨家庭、工厂、未知工具的稳定落地仍处于论文和早期产品化之间。

## 一句话理解

VLA 是把机器人动作接入多模态大模型的路线；它让语义泛化更强，但把 workload 引入 visual token、KV cache、action token、低延迟闭环和本体状态融合的组合压力。

## Workload Characterization

- 计算密度：VLM/LLM backbone 是主要计算；action head 相对小，但高频执行会放大端侧总负载。
- 访存模式：视觉 token、语言 token、robot state、历史观测和 KV cache 共同占用带宽与容量；多相机或 wrist camera 会增加 token 输入。
- 并行性：视觉编码和多相机输入可并行；autoregressive action decode 存在串行依赖；action chunk 可降低实时调用频率。
- 数据复用：同一视觉语言表征可复用给动作、任务判断、失败检测和解释；短时历史 cache 可跨控制周期复用。
- 量化敏感度：backbone 可低比特部署；action head、robot state embedding、末端姿态回归和 gripper 状态需谨慎验证。
- 瓶颈类型：端侧通常受 latency、memory capacity、KV cache bandwidth 和 batch=1 效率限制；训练侧受机器人数据吞吐与多模态对齐成本限制。
- 对硬件的核心需求：低 batch VLM 推理、连续控制低延迟、token/cache 管理、多传感器输入、动作解码和安全监控并行运行。

## 参考来源

- Zitkovich et al., `RT-2: Vision-Language-Action Models Transfer Web Knowledge to Robotic Control`, CoRL 2023 / arXiv:2307.15818，https://arxiv.org/abs/2307.15818，成熟度：VLA 范式代表，查证日期：2026-05-29。
- Kim et al., `OpenVLA: An Open-Source Vision-Language-Action Model`, arXiv:2406.09246，https://arxiv.org/abs/2406.09246，成熟度：开源研究模型，查证日期：2026-05-29。
- `pi0.5: a Vision-Language-Action Model with Open-World Generalization`, arXiv:2504.16054，https://arxiv.org/abs/2504.16054，成熟度：2025 前沿研究，查证日期：2026-05-29。
- NVIDIA, `GR00T N1: An Open Foundation Model for Generalist Humanoid Robots`, arXiv:2503.14734，https://arxiv.org/abs/2503.14734，成熟度：2025 产业研究原型，查证日期：2026-05-29。

## 旧版素材

- `/mnt/e/workload-wiki-old/04_机器人与VLA/VLA基础.md`
