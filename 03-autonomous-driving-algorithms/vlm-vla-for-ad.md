# VLM and VLA for Autonomous Driving

上级：[Autonomous Driving Algorithms](README.md)
相关：[VLA Workload](../06-chip-workload-analysis/vla-workload.md), [VLA Fundamentals](../04-robotics-and-vla/vla-fundamentals.md), [Transformer Workload](../06-chip-workload-analysis/transformer-workload.md)

## 这页在回答什么问题

这页回答 VLM/VLA 为什么进入自动驾驶算法路线。VLM 主要解决场景理解、推理、解释、驾驶问答；VLA 进一步把 vision/language token 接到 action 或 trajectory 输出，目标是可泛化的驾驶 policy。这是 03 章需联网核实的前沿方向，内容反映 2025-2026 的当前状态。

## 为什么要把大模型塞进车里：直觉与类比

传统感知是**闭卷考试**——只认训练时见过的固定类别。它在常规路况上很强，但碰到长尾就傻眼：前方交警在用手势指挥（和红灯矛盾，该听谁的？）、施工区的临时改道牌、侧翻露出底盘的异形卡车、有人举着"前方事故"的纸板。这些要么不属于任何已知类别（检测框不出来），要么需要"读懂语义再做判断"的推理能力（交警手势的含义），闭卷模型都给不了。

VLM 的直觉是**给车配一个会看图说话、会讲道理的副驾**。它带着 web-scale 的视觉语言知识上车，能开卷理解开放词汇的对象（"那是个交警"），能做常识推理（"交警指挥优先于信号灯"），还能把决策翻译成人能审查的语言理由（"我减速是因为右侧有孩子在追球"）。为什么这有效：语言/web 知识提供了闭集感知给不了的开放语义和推理先验——长尾之所以是长尾，恰恰是因为它在专用数据里罕见，但在大模型见过的互联网语料里并不罕见。

VLA 则更进一步——**让这个副驾不只动嘴，还直接动手**，把多模态上下文映射成 future trajectory 或 action token。但这里有个必须看清的张力：**会讲道理不等于会安全驾驶**。语言推理可能产生幻觉（一本正经地编一个不存在的理由），且大模型的 token decode 慢，未必满足车端几十毫秒的闭环延迟。所以 VLM/VLA 必须和几何、运动、控制约束结合，而不是单独决策。

## 部署的现实形态：慢-快双系统

理解这个方向当前怎么真正落地，关键是 **dual-system（慢-快双系统）** 架构，直觉是**一个深思熟虑但慢的副驾 + 一个反应快的司机**。DriveVLM 类路线让一个大 VLM 在较低频率上做慢推理（理解场景、识别长尾、给出高层决策意图），同时一个传统的快速 planner 在高频上做实时轨迹生成和安全兜底。慢系统负责"想明白"，快系统负责"及时做"——这正好化解了"大模型推理强但延迟高、安全 planner 快但不懂语义"的矛盾。这是 2025-2026 量产探索里最现实的形态，因为它不要求大模型满足实时闭环延迟。

```text
multi-view video + map + route + ego state + instruction
   -> visual tokens + language tokens + temporal tokens
   -> (慢) VLM reasoning / QA / explanation / risk    -> 高层意图
   -> (快) trajectory / waypoint / action decoder      -> 可执行轨迹
```

## 一句话理解

VLM/VLA 把自动驾驶从闭集类别识别推向开放语义理解和 action token 学习，用 web 知识补长尾、用语言给可审查理由；但它把 workload 引入大模型 token 爆炸、长上下文、KV cache、batch=1 低延迟解码的新约束，当前最现实的落地是"慢 VLM 推理 + 快 planner 执行"的双系统。

## 演进与未来方向（判断）

以下为判断，锚定 2025-2026 联网核实的真实工作。

演进脉络：从 DriveLM（把驾驶问答组织成 graph VQA）、DriveVLM（VLM 与 BEV/trajectory 联合）起步，到 Waymo 的 EMMA（端到端多模态、直接从传感器到规划）、2025 的 AutoVLA（自适应推理 + 强化微调直接出 action）、OpenDriveVLA（AAAI 2026，开源大 VLA，2D/3D 视觉 token 投到统一语义空间出轨迹），方向是**从"VLM 当解释/标注工具"走向"VLA 直接产出可部署 action"**。一篇 2025 末的综述（VLA for AD: Past, Present, and Future）已经在系统梳理这条线，说明它正从零散探索进入范式整理期。

我的几条判断。其一，**近期可量产的是双系统，而非车端纯 VLA**。车端实时跑一个完整大 VLA 出每一帧轨迹，延迟和功耗目前都不现实；慢-快双系统让大模型在可容忍的低频上提供语义和长尾判断，快 planner 保实时和安全——这会是未来 2-3 年的主流工程形态。其二，**语言中间步骤会被压缩**。早期 VLA 显式生成语言推理链再转 action，但语言 token 慢且引入幻觉，趋势是把推理隐式化、直接 visual-to-action，只在需要解释时才显式出语言（这与机器人 VLA 的演进同源，见 [VLA Fundamentals](../04-robotics-and-vla/vla-fundamentals.md)）。其三，**reasoning 的算力会变成可调档位**——AutoVLA 的"自适应推理"就是按场景难度决定推理深度，常规路况浅推理、长尾深推理。

对架构师，这个方向的 workload 含义非常硬，且正是 06 [VLA Workload](../06-chip-workload-analysis/vla-workload.md) 的核心：车端要在 batch=1 下跑 LLM/VLM decode（memory-bandwidth 和 KV cache 主导，见 [Transformer Workload](../06-chip-workload-analysis/transformer-workload.md) 的 decode 分析），多视角视觉 token 数巨大（visual token 爆炸压上下文），还必须**和传统安全 planner 并行运行**（双系统意味着芯片要同时承载一个大模型推理流和一个低延迟确定性 planner 流）。对 archax，这是一个"大模型 decode 工作点 + 实时 planner 工作点"并存的系统，关键资源是 KV cache 带宽与容量、以及两个异构工作流的并发隔离——而"自适应推理"意味着 reasoning 的 token 预算是个运行时可变量，必须按场景难度的分布而非平均值来评估最坏延迟。

## Workload Characterization

计算密度：vision encoder、projector、LLM/VLM decoder 是主计算；若输出 action token，autoregressive decode 影响 latency；双系统下慢 VLM 与快 planner 算力性格不同。

访存模式：visual token、language token、route token、历史 token、KV cache 需高带宽读写；多视角视觉 token 数大，多轮 reasoning 放大缓存。

并行性：视觉编码可并行；LLM token decode 串行性强；多候选 action sampling 可批并行；双系统的慢-快两流并发。

数据复用：同一视觉 token 服务问答、解释、风险识别、action head；KV cache 跨短时上下文复用。

量化敏感度：LLM 权重可低比特，但多模态 projector、关键安全 token、trajectory/action head 需保守验证（关乎安全且幻觉风险）。

瓶颈类型：车端常是 memory capacity、KV cache bandwidth、token latency；reasoning 深度自适应使最坏延迟由难场景决定；云端训练是多模态数据吞吐与长上下文训练。

对硬件的核心需求：低 batch LLM/VLM 推理、KV cache 管理、多模态 token 拼接、action head 低延迟输出，以及与传统安全 planner 并行运行的并发隔离——详见 [VLA Workload](../06-chip-workload-analysis/vla-workload.md)。

## 参考来源

- Sima et al., `DriveLM: Driving with Graph Visual Question Answering`, ECCV 2024 / arXiv:2312.14150。成熟度：VLM 数据与评测研究成熟。
- Tian et al., `DriveVLM: The Convergence of Autonomous Driving and Large Vision-Language Models`, CoRL 2024 / arXiv:2402.12289。成熟度：研究原型，慢-快双系统代表。
- Hwang et al. (Waymo), `EMMA: End-to-End Multimodal Model for Autonomous Driving`, arXiv:2410.23262。成熟度：前沿研究原型，端到端多模态。
- Zhou et al., `OpenDriveVLA: Towards End-to-end Autonomous Driving with Large Vision Language Action Model`, AAAI 2026 / arXiv:2503.23463。成熟度：2025-26 开源 VLA 代表，查证日期：2026-06-07。
- `AutoVLA: A Vision-Language-Action Model for End-to-End Autonomous Driving with Adaptive Reasoning and Reinforcement Fine-Tuning`, arXiv:2506.13757。成熟度：2025 前沿研究，自适应推理 + RL 微调，查证日期：2026-06-07。
- `Vision-Language-Action Models for Autonomous Driving: Past, Present, and Future`, arXiv:2512.16760。成熟度：2025 综述，方向梳理，查证日期：2026-06-07。
