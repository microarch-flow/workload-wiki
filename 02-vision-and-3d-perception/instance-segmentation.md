# Instance Segmentation

上级：[Vision and 3D Perception](README.md)
相关：[Object Detection](object-detection.md), [Semantic Segmentation](semantic-segmentation.md), [CNN Workload](../06-chip-workload-analysis/cnn-workload.md)

## 这页在回答什么问题

实例分割回答每个像素属于哪个对象实例。它叠加了 detection 的对象级区分和 segmentation 的像素级边界，对机器人抓取、遮挡对象分离、对象级 world state 很重要。这页重点是它引入的两个 workload 难点：动态实例数（带来动态 shape）和 per-instance/per-query 的 mask 处理。

## 为什么它有效：直觉与类比

先看它和语义分割差在哪。语义分割涂色不分个体——两只挨在一起的猫会被涂成同一片"猫色"。实例分割要进一步回答"这是第 1 只、那是第 2 只"，把 dense 像素组织成有身份的个体。难点正是个体数量事先不知道，这把"固定形状的 dense 输出"变成了"数量可变的对象集合"。

历史上两种思路，各自有清楚的直觉。**detect-then-segment**（Mask R-CNN）是**先框出每个个体，再在框内涂色**：先用检测找到每个对象的 ROI，把 ROI 区域对齐裁剪成固定大小（ROIAlign，如 14×14/28×28），再在这一小块上跑 mask head 画出该对象的轮廓。为什么有效：把"在全图分清所有个体"这个难题，拆成"先定位、再在干净的小框里二分前景背景"两个易题。代价是 ROI 的动态裁剪——框的数量和位置运行时才知道，对硬件是动态 shape。

**query-based**（Mask2Former）是**派固定数量的侦探，每个直接认领一个个体并画出它的整张轮廓**。它的精妙在 mask 怎么生成：每个 query 学出一个 embedding，把这个 embedding 和整张 pixel feature 图做点积，点积高的像素就属于这个 query 的 mask——也就是 `mask = query_embedding · pixel_feature`，一次 dense 乘法就把"哪些像素归这个个体"算出来了。为什么这统一而优雅：换 query 数量就能从语义分割（每类一个 query）平滑过渡到实例/全景分割，三个任务一套框架（见 [Semantic Segmentation](semantic-segmentation.md)）。代价是 query 数 × pixel feature 的 dense multiplication，以及和检测一样的 set-prediction 匹配。

## 计算结构与硬件瓶颈的分叉

```text
image -> backbone/neck -> object proposals or queries -> mask head -> instance masks
```

关键 workload 参数是**实例数量和 mask 分辨率**，且两条路线的硬件瓶颈不同。Mask R-CNN 对每个 proposal 提固定大小 ROI feature 再 upsample 出 mask，瓶颈在 ROIAlign 的动态裁剪/插值和随实例数变化的 batch——实例数动态意味着每帧计算量浮动，对静态调度的 NPU 是负载不均和动态 shape 问题。Mask2Former 用固定数量 query 出 mask embedding 再和 pixel feature dense 相乘，瓶颈在 query-mask multiplication 和高分辨率 mask 的处理——计算更规则（固定 query 数消除了动态 shape），但 mask 的高分辨率写回带宽和语义分割同源。

常见误解：实例分割比语义分割只是多了 instance id。实际上它引入对象数量动态变化、per-instance mask head 或 query matching，系统调度比纯 dense segmentation 复杂——动态性是它区别于语义分割的根本 workload 特征。

## 在 AD 和机器人中的意义

自动驾驶里实例分割不是所有量产主链路的必选，但能提供障碍物轮廓和遮挡分离。机器人里它更关键——抓取、分拣、桌面操作需要对象轮廓和实例边界来确定抓哪、从哪下手。

## 一句话理解

实例分割把 dense 像素组织成有身份的对象集合；它的 workload 同时背着高分辨率 mask（带宽）和动态实例数（动态 shape/负载不均），detect-then-segment 把动态性压在 ROIAlign，query-based 把它换成固定 query 的 dense mask 乘法。

## 演进与未来方向（判断）

以下为判断，锚定真实趋势。

主线和检测、语义分割完全一致：**query-based 的统一 set-prediction 正在吞并 detect-then-segment 和任务专用 head**。Mask2Former 已用一套 query 框架统一语义/实例/全景，检测也在走 query 化（见 [Object Detection](object-detection.md)）。我的判断是 2D 感知的终态是"一个共享 backbone + 一组 scene query + set prediction"，检测、实例分割、全景分割不再是独立模型而是同一 query 集合的不同读出。对架构师，这个收敛有个实在的好处：**固定数量的 query 消除了 detect-then-segment 的动态实例数问题**，把不规则的动态 shape 换成规则的固定 query 计算，对静态调度的 NPU 更友好；保留的难点是 query-mask dense multiplication 和高分辨率 mask 带宽。

第二条对机器人尤其相关：开放词汇/可提示分割（SAM 类）正让实例分割从"预定义类别"走向"任意对象提示分割"。这对机器人操作价值很大（抓一个训练时没见过的工具），但 workload 上引入 prompt encoder 和可能更大的模型。判断是机器人端会越来越依赖这类可提示分割做对象级感知，端侧要权衡其模型规模与实时性。对 archax，实例分割应按"固定 query 数 + mask 分辨率"参数化，把动态实例数是否存在作为 Interaction 轴上影响调度确定性的开关——这与检测的 NMS 同步点是同类的确定性考量。

## Workload Characterization

计算密度：backbone/neck 可 compute-bound；per-instance mask head 受实例数和 mask 分辨率影响，query-mask dense multiplication 随 query 数 × pixel 数增长。

访存模式：ROIAlign 的动态裁剪/插值是局部不规则访问；query-mask 乘法是规则 dense 访问但读全图 pixel feature；高分辨率 mask 写回是带宽压力（与语义分割同源）。

并行性：实例、query、mask pixel 可并行；detect-then-segment 的实例数动态导致 batch 内负载不均，query-based 的固定 query 数并行更均衡。

数据复用：共享 backbone/neck feature；Mask R-CNN 每实例重复读局部 ROI feature；Mask2Former 的 pixel feature 被所有 query 复用。

量化敏感度：backbone/head 可低比特；mask boundary、ROIAlign 插值需谨慎（边界对量化敏感）。

瓶颈类型：机器人在线操作常 latency-bound；高实例数场景 detect-then-segment 易 dispatch/负载不均-bound；query-based 主要 mask 带宽-bound。

对硬件的核心需求：动态 query/proposal 调度、ROIAlign/mask 操作、query-mask multiplication、高分辨率 upsample 与写回优化、低延迟 postprocess——query 路线可消除动态实例数带来的不规则调度，详见 [CNN Workload](../06-chip-workload-analysis/cnn-workload.md)。

## 参考来源

- He et al., `Mask R-CNN`, ICCV 2017, arXiv:1703.06870。成熟度：已落地，detect-then-segment 代表。
- Cheng et al., `Masked-attention Mask Transformer for Universal Image Segmentation (Mask2Former)`, CVPR 2022, arXiv:2112.01527。成熟度：已落地，query-based 统一分割。
- Kirillov et al., `Segment Anything (SAM)`, ICCV 2023, arXiv:2304.02643。成熟度：已落地开源，可提示/开放词汇分割，对机器人对象级感知意义大。
