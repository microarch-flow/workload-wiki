# Contrastive Learning

上级：[Foundation Model Components](README.md)
相关：[JEPA and Self-supervised Representation](jepa-and-self-supervised.md), [CNN Backbone](cnn-backbone.md), [Vision Transformer Backbone](vision-transformer-backbone.md)

## 这页在回答什么问题

这页回答 Contrastive Learning 为什么是视觉表示预训练的重要起点，以及它对 workload 的意义——注意它主要是**训练范式**，对推理 workload 的影响是间接的（通过 backbone 质量、token 表示、下游微调成本和模型规模选择）。这里不展开 loss 推导，关注"拉近正样本、拉远负样本"如何改变训练系统和下游 backbone 的使用方式。

## 为什么它有效：直觉与类比

Contrastive learning 的直觉是**不靠老师给标准答案，靠"找不同"来学**。给模型看同一个东西的两张不同照片（同一张图做两次不同裁剪、调色、加噪——这是两个"视图"），告诉它"这俩是一伙的，要在表示空间里靠近"；再随便找些别的东西，告诉它"这些是外人，要推远"。反复做下来，模型被迫学会：什么变了（光照、裁剪、角度）是无关紧要的表面，什么没变（这是一只猫）才是本质。

为什么这能学到有用的表征：它把"理解一个物体"偷换成了一个不需要人工标签的代理任务——"在各种干扰下还能认出这是同一个东西"。要做到这点，模型没法只记纹理像素（裁剪一变就对不上了），必须抓住裁剪、调色都破坏不了的高层语义。于是在没有任何"这是猫"标注的情况下，它学出了一个把猫和狗自然分开的表示空间。对自动驾驶和机器人，这点尤其值钱——原始视频和传感器数据海量，但高质量标注昂贵，contrastive pretraining 让海量无标注数据先把 encoder 喂出一个好底子。

这里有个关键的工程含义藏在"找外人"里：要让"推远外人"这件事有意义，每一步得有足够多的外人（负样本）作对比。负样本太少，模型偷懒就能区分，学不到细腻的语义。这个"需要大量负样本"的诉求，直接决定了 contrastive 训练的系统特征——要么靠超大 batch，要么靠队列/memory bank，下一节展开。

常见误解：Contrastive Learning 是一个推理侧模型组件。实际上它是训练范式，推理时通常只留 encoder，runtime workload 仍由 CNN/ViT backbone 决定。

## 从 SimCLR / MoCo 到自蒸馏：负样本问题的两种解法

整条演进线的暗线是"怎么搞到足够多负样本而不被显存压垮"。SimCLR 的解法是**硬上大 batch**：两个视图各编码，在 batch 内构造相似度矩阵——batch size `B` 时要比较约 `2B×2B` 的 embedding 相似度，batch 越大负样本越多，于是它需要极大 batch（数千）和强数据增强，多卡训练还要 all-gather 各设备的 embedding 才能凑够负样本。MoCo 的解法是**用队列攒历史负样本**：维护一个 memory queue 存过去若干 batch 的 embedding 当负样本，再用 momentum encoder（缓慢滑动更新的编码器）保证队列里的旧 embedding 不至于和当前编码器差太远——这样不靠极大 batch 也能有大量负样本，代价是队列读写和 momentum encoder 的维护。

BYOL、SimSiam、DINO 更进一步，**干脆弱化甚至取消显式负样本**，转向 bootstrap/self-distillation（让学生网络预测教师网络的输出，靠 stop-gradient 和结构设计防止塌缩）。DINO 对 ViT 尤其重要——它让自监督视觉 token 形成强语义结构，直接影响后续 ViT backbone、VLM、VLA 的 vision encoder 质量。这条线的演进说明：表示预训练的重点从"如何构造负样本"逐步转向"如何让模型学到稳定、有语义结构的 embedding"。

## 对自动驾驶和机器人的实际作用

自动驾驶 camera 数据规模大、标注贵，contrastive/self-supervised pretraining 能改善 image encoder 初始化，降低检测、分割、BEV、E2E 的标注依赖。机器人同样有大量无标注视频和操作观察，对比式/自蒸馏表征为 VLA 提供更稳的 vision encoder，让 language/action head 不必从头学底层视觉结构。但要清楚它的边界：它给的是 representation foundation，不解决 closed-loop control，也不直接给 world dynamics——后两者是 [VLA Fundamentals](../04-robotics-and-vla/vla-fundamentals.md) 和 [World Model Fundamentals](../05-world-model-and-generative/world-model-fundamentals.md) 的事。

## 一句话理解

Contrastive learning 用"在各种干扰下认出同一个东西"这个无标注代理任务，把海量无标注数据变成强 backbone；它主要改变上游训练 workload（大 batch / 队列 / 跨设备通信）和下游 backbone 质量，几乎不直接增加端侧推理算子。

## 演进与未来方向（判断）

以下为判断，锚定真实趋势。

第一，**自监督视觉表征正在收敛到少数几个强 backbone，并成为多模态系统的默认 vision tower**。DINOv2 把自蒸馏路线推到生产可用的通用特征，VLM/VLA 越来越多直接拿它当冻结的视觉前端。我的判断是，未来下游系统不再自己从头训 vision encoder，而是复用这类自监督大 backbone——这把架构师该关心的事从"训练范式"彻底推回"推理侧 backbone 的 workload"：你部署时面对的是一个固定的、可能偏大的 ViT/CNN encoder，量化与 layout 由它决定，contrastive loss 本身不进硬件预算。

第二，**对比式与预测式（JEPA 类）的边界在模糊**。纯对比依赖负样本和大 batch，工程上重；[JEPA](jepa-and-self-supervised.md) 这类在表示空间做预测的方法绕开了负样本构造，且更贴近 World Model 所需的"可预测结构"。判断是视觉自监督的重心会从"对比"继续向"预测表征"迁移，对比学习更多作为历史主线和某些对齐任务（如 CLIP 式图文对比）保留。对架构师的实际含义很务实：不必为 contrastive 训练专门建硬件模型，但要预期它会持续把更强、更大的 encoder 推进下游系统——端侧真正要扛的，是这些预训练范式产出的 backbone，而非范式本身。

## Workload Characterization

计算密度：训练侧由双分支 encoder 和 embedding similarity 主导，compute-heavy；推理侧回到普通 CNN/ViT encoder，由 backbone 决定——这是 contrastive 的本质特征：训练重、推理无额外算子。

访存模式：训练需保存多视图 activation、embedding、队列或大 batch 表示；MoCo 的 queue 是额外读写源；推理侧无额外 contrastive 访存。

并行性：训练可沿 batch/data parallel 扩展，但 embedding all-gather（SimCLR 凑负样本）和队列维护引入跨设备通信；推理侧按 backbone 并行。

数据复用：encoder weight 在两视图间共享；momentum encoder/queue 提供历史表示复用，但只存在于训练侧。

量化敏感度：训练通常 FP16/BF16（embedding 相似度和对比目标对数值敏感）；推理侧量化完全由 backbone 决定。

瓶颈类型：训练侧可能同时 compute + communication（all-gather）+ memory-capacity（大 batch / queue）bound；端侧推理瓶颈不由 contrastive objective 决定。

对硬件的核心需求：云端训练需要高吞吐矩阵/卷积、跨设备通信（NVLink/PCIE，凑负样本的 all-gather）、大 batch memory；端侧只继承 encoder 的 workload——架构师做端侧 NPU 时不必为 contrastive loss 建模，但要预判它推动的 encoder 规模。

## 参考来源

- He et al., `Momentum Contrast for Unsupervised Visual Representation Learning (MoCo)`, CVPR 2020, arXiv:1911.05722。成熟度：已落地，队列 + momentum encoder。
- Chen et al., `A Simple Framework for Contrastive Learning of Visual Representations (SimCLR)`, ICML 2020, arXiv:2002.05709。成熟度：已落地，大 batch 对比。
- Caron et al., `Emerging Properties in Self-Supervised Vision Transformers (DINO)`, ICCV 2021, arXiv:2104.14294。成熟度：已落地，自蒸馏 + ViT。
- Oquab et al., `DINOv2: Learning Robust Visual Features without Supervision`, arXiv:2304.07193。成熟度：已落地开源，生产级通用自监督特征。
