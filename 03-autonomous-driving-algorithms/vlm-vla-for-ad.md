# VLM and VLA for Autonomous Driving

上级：[Autonomous Driving Algorithms](README.md)
相关：[VLA Workload](../06-chip-workload-analysis/vla-workload.md), [Contrastive Learning](../01-foundation-model-components/contrastive-learning.md)

## 这页在回答什么问题

这页回答 VLM / VLA 为什么会进入自动驾驶算法路线。VLM 主要解决 scene understanding、reasoning、解释和驾驶问答；VLA 进一步把 language/vision tokens 接到 action 或 trajectory 输出上，目标是形成可泛化的驾驶 policy。

## 从 VLM 到 VLA

```text
multi-view video + map + route + ego state + instruction
   ->
visual tokens + language tokens + temporal tokens
   ->
reasoning / QA / explanation / risk assessment
   ->
trajectory / waypoint / action token
```

VLM 更像“驾驶语义和推理层”：理解路况、解释意图、回答 why/what-if。VLA 更接近“驾驶动作层”：把多模态上下文映射为 future action。

## VLM 的价值

VLM 在自动驾驶中的直接价值包括：

- 场景解释：把模型决策转换为可审查的语言理由。
- 长尾理解：对施工、交警、临时标志、异形车辆等开放词汇对象更友好。
- 数据闭环：辅助生成场景标签、失败原因、检索描述和仿真 prompt。
- 人机接口：接受 route instruction、乘客偏好、远程协助文本。

但 VLM 的 language reasoning 不天然等于安全驾驶。它需要与几何、运动、控制约束结合。

## VLA 的变化

VLA 把 action 纳入 token 或连续 decoder。近期路线开始探索用大模型统一 perception、reasoning 和 trajectory generation，例如 DriveVLM、DriveLM、EMMA、AutoVLA 等。

关键问题不是“能不能输出方向盘角度”，而是：

- action token 是否足够表达连续驾驶控制；
- language prior 是否会引入幻觉；
- closed-loop latency 是否满足车端约束；
- 模型能否在 rare event 上稳定优于专用 planner。

## 成熟度判断

- 成熟：VLM 作为解释、数据标注、驾驶问答和辅助评测工具。
- 发展中：VLM 与 BEV/trajectory decoder 的联合训练。
- 前沿：VLA 直接输出可部署 action，尤其是 2025-2026 的 AutoVLA、EMMA 类路线，仍以研究和原型为主。

## 一句话理解

VLM/VLA 把自动驾驶从封闭类别识别推向开放语义和 action token 学习，但它把 workload 引入了大模型 token、长上下文、多模态对齐和低延迟解码的新约束。

## Workload Characterization

- 计算密度：vision encoder、projector、LLM/VLM decoder 是主计算；若输出 action token，autoregressive decode 会影响 latency。
- 访存模式：visual token、language token、route token、历史 token 和 KV cache 需要高带宽读写；多轮 reasoning 会放大缓存。
- 并行性：视觉编码可并行；LLM token decode 串行性强；多候选 action sampling 可批并行。
- 数据复用：同一视觉 token 可服务问答、解释、风险识别和 action head；KV cache 可跨短时上下文复用。
- 量化敏感度：LLM 权重可用低比特，但多模态 projector、关键安全 token、trajectory/action head 需要保守验证。
- 瓶颈类型：车端常是 memory capacity、KV cache bandwidth 和 token latency；云端训练常是多模态数据吞吐和长上下文训练。
- 对硬件的核心需求：低 batch LLM/VLM 推理、KV cache 管理、多模态 token 拼接、action head 低延迟输出、与传统安全模块并行运行。

## 参考来源

- Sima et al., `DriveLM: Driving with Graph Visual Question Answering`, ECCV 2024 / arXiv:2312.14150，https://arxiv.org/abs/2312.14150，成熟度：VLM 数据与评测研究成熟，查证日期：2026-05-29。
- Tian et al., `DriveVLM: The Convergence of Autonomous Driving and Large Vision-Language Models`, CoRL 2024 / arXiv:2402.12289，https://arxiv.org/abs/2402.12289，成熟度：研究原型，查证日期：2026-05-29。
- Waymo Research, `EMMA: End-to-End Multimodal Model for Autonomous Driving`, arXiv:2410.23262，https://arxiv.org/abs/2410.23262，成熟度：前沿研究原型，查证日期：2026-05-29。
- `AutoVLA: A Vision-Language-Action Model for End-to-End Autonomous Driving with Adaptive Reasoning and Reinforcement Fine-Tuning`, arXiv:2506.13757，https://arxiv.org/abs/2506.13757，成熟度：2025 前沿研究，查证日期：2026-05-29。

## 旧版素材

- `/mnt/e/workload-wiki-old/03_自动驾驶算法路线/VLM_VLA_for_AD.md`
