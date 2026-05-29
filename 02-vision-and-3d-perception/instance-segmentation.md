# Instance Segmentation

上级：[Vision and 3D Perception](README.md)
相关：[Object Detection](object-detection.md), [Semantic Segmentation](semantic-segmentation.md), [CNN Workload](../06-chip-workload-analysis/cnn-workload.md)

## 这页在回答什么问题

实例分割回答每个像素属于哪个对象实例。它结合 detection 的对象级区分和 segmentation 的像素级边界，对机器人操作、遮挡对象分离和对象级 world state 很重要。

## 计算结构

典型 detect-then-segment 链路是：

```text
image -> backbone/neck -> object proposals or queries -> mask head -> instance masks
```

Mask R-CNN 类方法先检测对象，再为每个 ROI 预测 mask。Bottom-up 方法先做像素/embedding，再 grouping 成实例。Query-based 方法用 object queries 直接预测 mask，与 Transformer scene representation 更一致。

实例分割的关键 workload 参数是实例数量和 mask 分辨率。实例数动态变化会带来动态 shape；mask head 需要较高分辨率 feature；ROIAlign、mask upsample、query-mask multiplication 都会引入额外 memory 和调度成本。

Mask R-CNN 常见做法是对每个 proposal 提取固定大小 ROI feature，例如 14x14 或 28x28，再用 mask head 输出 per-instance mask。Mask2Former/Query-based segmentation 则用固定数量 queries 预测 mask embedding，再和 pixel feature 做乘法得到 mask logits。前者有 ROI 动态裁剪，后者有 query-mask dense multiplication，硬件瓶颈不同。

## 在 AD 和机器人中的意义

自动驾驶中实例分割不是所有量产主链路的必选项，但它能提供障碍物轮廓和遮挡分离。机器人中它更关键，因为抓取、分拣、桌面操作需要对象轮廓和实例边界。

常见误解：实例分割比语义分割只是多了 instance id。实际上，它引入对象数量动态变化、per-instance mask head 或 query matching，系统调度比纯 dense segmentation 更复杂。

## 一句话理解

实例分割把 dense pixels 组织成对象级 masks；它的 workload 同时具有高分辨率 mask、动态实例数和 query/proposal 处理。

## Workload Characterization

- 计算密度：backbone/neck 可 compute-bound；per-instance mask head 常受实例数和 mask resolution 影响。
- 访存模式：ROI/mask feature 裁剪、upsample、query-mask 乘法会增加不规则或局部 feature 访问。
- 并行性：实例、query、mask pixel 可并行；实例数动态导致 batch 内负载不均。
- 数据复用：共享 backbone/neck feature；每个实例 mask head 重复读取局部 feature。
- 量化敏感度：backbone/head 可低比特；mask boundary 和 ROIAlign/interpolation 需要谨慎。
- 瓶颈类型：机器人在线操作常 latency-bound；高实例数场景可能 memory/dispatch-bound。
- 对硬件的核心需求：动态 query/proposal 调度、ROIAlign/mask 操作、query-mask multiplication、upsample、低延迟 postprocess。

## 参考来源

- He et al., `Mask R-CNN`, ICCV 2017, arXiv:1703.06870。
- Cheng et al., `Masked-attention Mask Transformer for Universal Image Segmentation`, CVPR 2022, arXiv:2112.01527。

## 旧版素材

- `/mnt/e/workload-wiki-old/02_视觉与3D感知/实例分割/实例分割总览.md`
