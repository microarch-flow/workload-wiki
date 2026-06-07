# JEPA and Self-supervised Representation

上级：[Foundation Model Components](README.md)
相关：[Contrastive Learning](contrastive-learning.md), [World Model Fundamentals](../05-world-model-and-generative/world-model-fundamentals.md), [World Model Workload](../06-chip-workload-analysis/world-model-workload.md)

## 这页在回答什么问题

这页回答 JEPA 为什么是从视觉自监督走向 World Model 的关键中间层。它的重点不是重建像素，而是**预测高层表征**——这一字之差让它比普通 masked reconstruction 更接近"学习世界中可预测的结构"，也让它在 workload 上避开了笨重的像素 decoder。看懂这页，才能理解 05 章的 latent world model 为什么在端侧比视频生成式 world model 更可行。

## 为什么它有效：直觉与类比

JEPA 的直觉是**预测要点，不预测原话**。给你看一段视频的前半段，遮住后半段，有两种考法。一种（MAE/像素重建）要求你把后半段每一个像素都画出来——草叶的纹理、镜头的噪点、墙面的反光，全都要还原。另一种（JEPA）只要求你说出"后半段大概会发生什么"——那辆车会继续往左开、那个杯子会被拿起来——在语义/表示层面预测，不管像素长什么样。

为什么第二种更聪明：像素级预测逼模型把大量算力浪费在**根本不可预测、也不重要的细节**上。草叶下一帧的确切纹理、传感器的随机噪点，这些既猜不准也没用，但像素重建会因为它们贡献了大部分像素而拼命去拟合，结果模型学了一脑子表面统计。JEPA 在表示空间预测，等于先用一个 target encoder 把"要预测的东西"抽象成高层 embedding（已经滤掉了纹理噪点），再让模型预测这个 embedding——于是算力被集中到**可预测的结构**上：物体、运动、空间关系、物理趋势。而这些恰恰是自动驾驶、机器人、World Model 真正关心的东西。换句话说，JEPA 的有效性来自一个朴素判断——别让模型为它猜不准也用不上的细节买单。

机制上这带来一个不对称结构：context encoder 编码看得见的部分，predictor 据此预测被遮部分的表征，另有一个 target encoder（通常是 context encoder 的滑动平均）给出被遮部分的"真实"表征作监督。训练让预测表征逼近目标表征。常见误解：JEPA 就是另一种 masked autoencoder。实际上 MAE 的目标是恢复输入信号本身，JEPA 的目标是在 embedding 空间学上下文到目标的语义预测关系——预测对象不同，学到的东西也不同。

## I-JEPA、V-JEPA 与 V-JEPA 2：从图像到可规划的视频表征

I-JEPA 把这套思想用于图像，证明不靠像素重建也能学出高质量视觉表示。V-JEPA 扩展到视频，用时空上下文预测缺失或未来片段的表征。V-JEPA 2（2025）进一步把自监督视频表征和 physical world understanding、prediction、planning 连起来，并展示了 action-conditioned latent world model 的机器人规划用法——这是从"表征学习"跨到"可用于决策的 world model"的关键一步。V-JEPA 2.1（2026）继续强化 dense video/image feature，更适合作 perception、tracking、segmentation 等密集视觉任务的通用表征源。这些仍应理解为研究系统和开源模型阶段，不是量产机器人方案。

量级上，视频 JEPA 的输入不再是单图而是多 frame 或 tubelet token，history window、frame resolution、patch/tubelet size 共同决定 token 数，训练成本接近 video foundation model 而非普通 image 自监督。这条线对本 wiki 关键，因为 World Model 要预测未来但不必预测像素级未来——对芯片 workload 而言，latent representation prediction 可能比 raw video generation 更适合端侧风险评估和短 horizon planning。

## 对 World Model 的意义：为什么是分水岭

World Model 的预测对象可以是 pixel video、latent state、BEV、occupancy、action-conditioned future。JEPA 强调在表示空间预测，给 latent world model 提供了思想基础，也划出一条 workload 分水岭：如果预测对象是高维像素，workload 被 decoder 和视频生成细节主导（重、慢，见 [Diffusion for World Model](../05-world-model-and-generative/diffusion-for-world-model.md)）；如果预测对象是 latent 表征，计算集中在 encoder、predictor、latent dynamics 上，**没有笨重的像素 decoder**。后者更可能服务自动驾驶/机器人里的 planning、risk evaluation、state rollout——这是 [World Model Workload](../06-chip-workload-analysis/world-model-workload.md) 区分两类 world model 成本的核心依据。

## 一句话理解

JEPA 把自监督从"对比或重建输入"推进到"在表示空间预测高层表征"，靠"不为不可预测的细节买单"获得效率与语义；它是理解 latent world model 的桥梁——预测 latent 而非像素，让 world model 的 workload 从重 decoder 的生成式，转向 encoder + predictor + latent rollout 的轻量式。

## 演进与未来方向（判断）

以下为判断，锚定真实趋势。

第一，**world model 正在沿 JEPA 划出的这条线分叉成两支，且端侧会倒向 latent-predictive 一支**。一支是像素生成式（video diffusion 类，逼真但要重 decoder、多步采样，成本高，见 [Diffusion Models](diffusion-models.md)）；另一支是表征预测式（JEPA 类，在 latent 空间 rollout，无像素 decoder）。我的判断是，车端/机器人端的 planning 和 risk evaluation 会主要采用 latent-predictive——理由直接：决策需要的是"未来会怎样"的语义判断，不是逐像素的逼真画面，而省掉像素 decoder 和多步去噪能省掉数量级的算力和延迟。像素生成式更多留在云端仿真、数据生成、可视化。

第二，这条判断落到硬件上很具体：**latent world model 的 workload 形态 = 视觉 encoder（编码当前观测）+ predictor（GEMM 为主）+ action-conditioned latent state rollout**，没有大生成 decoder，但 rollout 引入按时间步推进的 latent state 更新——这和 [Mamba and SSM](mamba-and-ssm.md) 的状态递推、[Diffusion Models](diffusion-models.md) 的多步迭代在硬件上同属"带状态的迭代推理"，而非一次性 forward。对 archax，latent world model 应建模为"encoder（compute-bound）+ predictor + 迭代 latent rollout（带状态、latency 敏感）"的复合工作点，其 rollout 步是 Interaction 轴上必须显式建模的迭代维度。把这条提前想清楚，是因为 V-JEPA 2 这类工作正把 latent world model 从概念推向研究落地，端侧芯片需要预判它的算子构成——详见 [World Model Workload](../06-chip-workload-analysis/world-model-workload.md)。

## Workload Characterization

计算密度：encoder/predictor 多为 CNN/ViT/Transformer，训练侧 compute-heavy；latent prediction 比 pixel reconstruction 省掉重 decoder，推理侧算力压力显著更低——这是 JEPA 相对生成式的核心 workload 优势。

访存模式：训练需保存 context/target 表示和视频 token；视频版本的时空 token 放大 activation memory；在线 latent rollout 反复读写 latent state（类似但远轻于 diffusion 的 latent 往返）。

并行性：image JEPA 可沿 batch/patch 并行；video JEPA 可沿 batch、spatial token、部分 temporal chunk 并行，但 action-conditioned 长 horizon rollout 沿时间有依赖。

数据复用：context representation、target embedding、latent state 可复用；world model 场景中压缩状态可跨 rollout 步复用（恒定大小，类似 SSM 的便签）。

量化敏感度：推理侧 encoder/predictor 可尝试低比特；训练侧及表征预测目标通常需 FP16/BF16 保稳定（表示空间的目标对数值敏感）。

瓶颈类型：训练侧多为 compute + activation-memory；在线 latent world model 转为 latency + state-cache bound（rollout 的迭代性质），而非生成式的 capacity-bound。

对硬件的核心需求：高效 video/ViT encoder、latent state buffer、predictor GEMM、temporal token 管理、action-conditioned 迭代 state update——注意这套需求里**没有**大生成 decoder，这正是它相对视频生成式 world model 的端侧优势，详见 [World Model Workload](../06-chip-workload-analysis/world-model-workload.md)。

## 参考来源

- Assran et al., `Self-Supervised Learning from Images with a Joint-Embedding Predictive Architecture (I-JEPA)`, CVPR 2023, arXiv:2301.08243。成熟度：已落地开源，图像 JEPA 出处。
- Bardes et al., `Revisiting Feature Prediction for Learning Visual Representations from Video (V-JEPA)`, 2024, arXiv:2404.08471。成熟度：已落地开源，视频 JEPA。
- Assran et al., `V-JEPA 2: Self-Supervised Video Models Enable Understanding, Prediction and Planning`, 2025, arXiv:2506.09985。成熟度：研究系统，action-conditioned latent world model 的代表。
- Mur-Labadia et al., `V-JEPA 2.1: Unlocking Dense Features in Video Self-Supervised Learning`, 2026, arXiv:2603.14482。成熟度：研究系统，强化 dense feature。
