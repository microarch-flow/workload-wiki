# Latent World Model

上级：[World Model and Generative Intelligence](README.md)
相关：[World Model Fundamentals](world-model-fundamentals.md), [JEPA and Self-supervised](../01-foundation-model-components/jepa-and-self-supervised.md), [Mamba and SSM](../01-foundation-model-components/mamba-and-ssm.md), [World Model Workload](../06-chip-workload-analysis/world-model-workload.md)

## 这页在回答什么问题

这页回答 latent World Model 为什么是最可能上端侧的一支：它在压缩 latent 空间里 rollout，不预测像素、没有重 decoder、不跑多步采样，于是把 World Model 的单步成本砍掉一两个数量级。重点是讲清"省掉像素 decoder"具体省在哪、代价（可解释性弱）出在哪，以及它和 JEPA、Dreamer、Mamba 状态递推的同构关系，而不是 RL 的训练细节。

## 为什么它有效：直觉与类比

latent World Model 的直觉是**下棋时在脑子里推演局面，而不是把每一步都画成照片**。高手算棋只在抽象的"局面"层面往下想——这步之后我子力占优、那条线会被将——根本不去想象棋子木纹的光影。latent World Model 就是这么做：把观测压成一个低维 latent state（局面），dynamics 在 latent 上递推 `z_t, action_t -> z_{t+1}`，全程不回到像素。对应到机制，计算集中在 encoder + predictor + latent 递推三件事上，**没有把 latent decode 回高分辨率像素的那个笨重 decoder**，也没有 diffusion 的几十步采样。

为什么这是省算的关键而不只是小优化：像素 decoder 和多步去噪恰是视频 World Model 最重的两块（见 [Video World Model](video-world-model.md)）。把预测对象从像素换成 latent，等于一次砍掉这两块——single-step 从"跑几十步的大 DiT + decoder"塌缩成"一个小网络的 forward"。对应到机制，这正是 [JEPA](../01-foundation-model-components/jepa-and-self-supervised.md) 的主张落到 World Model 上：别为猜不准也用不上的像素细节买单，只预测可决策的结构。也正因如此，latent 是唯一能在端侧承受 `candidate × horizon` 那个乘法的表示——单步够轻，乘上 64 个 candidate 和几十步 horizon 才跑得动。

代价藏在"压缩"里：latent 可解释性弱。如果编码时把小障碍、接触边界、交通灯状态压丢了，planner 很难发现——它看到的是一团抽象 state，不是能逐项检查的 occupancy。对应到机制，latent 的信息瓶颈是把它做轻的原因，也是它单独用不安全的原因，这就是为什么工程系统常把 latent dynamics（长时序预测）和 BEV/occupancy（几何安全约束）并用。

## 基本结构与 rollout 的状态递推

```text
observation -> encoder -> latent state z_t
   -> latent dynamics: (z_t, action_t) -> z_{t+1}   [反复迭代 H 步]
   -> reward / value / policy / 轻 decoder / risk head
```

Dreamer 系是典型：学 latent dynamics，再在 imagined rollout 里训练 policy（DreamerV3 在数百维 latent + 离散随机状态上递推）。量级上，latent state 是几百到几千维的小张量，单步递推是个小 GEMM 或 recurrent cell；这让端侧可以高频跑多 candidate rollout——同一 observation encoding 编码一次，复用给所有 candidate（`encode once → rollout many`），每个 candidate 沿 horizon 迭代。

这里有个对硬件最重要的观察：latent rollout 的"沿时间迭代、维护一个跨步驻留的 state"，和 [Mamba 的状态递推](../01-foundation-model-components/mamba-and-ssm.md)、[Diffusion 的多步迭代](../01-foundation-model-components/diffusion-models.md) 同属"带状态的迭代推理"这条主线。它不是一次性 forward，而是 state 常驻 + 逐步更新；rollout 内部沿时间严格依赖（z_{t+1} 依赖 z_t）是并行断点，跨 candidate 才能并行。archax 里 horizon 步数与 candidate 数是 Interaction 轴上必须显式扫描的两个维度。

## 与 JEPA 的关系：从表征学习走到可行动

JEPA 类方法也在 representation space 预测未来而非重建像素，给 latent World Model 提供了思想基础。关键的跨越发生在 V-JEPA 2（arXiv:2506.09985）：它在 1M+ 小时视频上自监督预训练表征，其 action-conditioned 变体 V-JEPA 2-AC 仅用 62 小时 Droid 机器人视频微调，就能预测"动作条件下的未来 latent"，再配 CEM 采样 + receding horizon 做实机规划——官方称推理比像素生成式的 Cosmos 快约 30×。这把 latent prediction 从"学表征"真正推到了"可行动的 world model"，也用一个具体数字佐证了 latent 路线相对像素生成的端侧优势。

## 一句话理解

latent World Model 在压缩状态里 rollout，用牺牲可解释性换取效率——省掉像素 decoder 与多步采样，单步成本降一两个数量级，是唯一能在端侧扛住 `candidate × horizon` 乘法的表示；它的 rollout 与 Mamba 状态递推、Diffusion 多步迭代同属带状态的迭代推理。

## 演进与未来方向（判断）

以下为判断，锚定 2025-2026 联网核实的真实工作。查证日期：2026-06-07。

第一，**latent World Model 是端侧 World Model 最现实的落点，2025-2026 正从概念走向研究落地**。V-JEPA 2-AC 用极少交互数据 + 无像素 decoder 的 latent 预测做实机规划，且明确比像素生成快约一个量级，正是端侧需要的形态。我的判断是：车端/机器人端的 planning 与 risk evaluation 会主要走 latent-predictive——决策要的是"未来会怎样"的语义判断而非逐像素画面，省掉 decoder 和采样能省掉数量级的算力与延迟。但近期更可能是"轻量 latent dynamics 做短 horizon 风险评估"，而非完整 latent imagination 闭环控制，因为可验证安全仍未解决。

第二，这条判断落到硬件上很具体：**latent World Model 的 workload 形态 = 视觉 encoder（compute-bound）+ predictor（GEMM 为主）+ action-conditioned latent rollout（带状态、latency 敏感、无大 decoder）**。它的硬件画像和视频 World Model 几乎相反——不要巨大 HBM 装 video latent，而要小张量高并发 + state cache 常驻 SRAM + 低延迟逐步更新。对 archax，应建模为"encoder + predictor + 迭代 latent rollout"的复合工作点，rollout 步是 Interaction 轴的迭代维度，candidate 数是并发维度；其状态递推这一类原语和 Mamba/SSM 共享需求，是相对今天纯 GEMM/attention NPU 的关键增量。这条直接衔接 06 [World Model Workload](../06-chip-workload-analysis/world-model-workload.md) 对 latent dynamics 的分析。

## Workload Characterization

计算密度：encoder 可能较重（ViT/CNN，compute-bound），latent dynamics 单步是几百到几千维的小网络（远轻于 video decoder）；总计算被 candidate × horizon 放大，但底数小到端侧可承受——这是 latent 相对生成式的核心 workload 优势。

访存模式：主要缓存 latent state、action candidate、history context、value/policy head；latent state cache 是低维小张量，容量远低于 video latent，可常驻 SRAM。

并行性：candidate action、imagined trajectory、environment batch 可高度并行；单条 trajectory 内部沿时间递推严格依赖，是并行断点（与 Mamba 状态递推同构）。

数据复用：同一 observation encoding 复用于所有 candidate rollout（`encode once → rollout many`）；latent state 可在 policy/value/world model head 间共享。

量化敏感度：latent dynamics 可尝试低比特；value/risk/collision 相关 latent 维度的误差需校验，且误差沿 rollout 累积，长 horizon 更敏感。

瓶颈类型：训练侧是 rollout 数与反向传播显存；端侧推理是 candidate 数、horizon 与实时预算（latency + state-cache bound），而非视频生成的 capacity-bound。

对硬件的核心需求：小张量高并发、state cache 常驻 SRAM、低延迟 recurrent/Transformer dynamics、多 rollout 调度、状态递推原语支持（与 [Mamba and SSM](../01-foundation-model-components/mamba-and-ssm.md) 共享）——注意这套需求里没有大生成 decoder，详见 [World Model Workload](../06-chip-workload-analysis/world-model-workload.md)。

## 参考来源

- Ha and Schmidhuber, `World Models`, NeurIPS 2018 / arXiv:1803.10122。成熟度：经典基础，latent world model 起点。
- Hafner et al., `Mastering Diverse Domains through World Models (DreamerV3)`, arXiv:2301.04104。成熟度：成熟，latent dynamics + 想象中训练 policy。
- Assran et al., `V-JEPA 2: Self-Supervised Video Models Enable Understanding, Prediction and Planning`, 2025, arXiv:2506.09985。成熟度：研究系统，action-conditioned latent world model 实机规划，查证日期：2026-06-07。
- LeCun, `A Path Towards Autonomous Machine Intelligence`, 2022。成熟度：概念框架，latent-predictive world model 的纲领性主张。
</content>
