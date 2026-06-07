# Object Detection

上级：[Vision and 3D Perception](README.md)
相关：[CNN Backbone](../01-foundation-model-components/cnn-backbone.md), [BEV Perception](bev-perception.md), [CNN Workload](../06-chip-workload-analysis/cnn-workload.md)

## 这页在回答什么问题

目标检测回答图像或场景中有哪些对象、在哪里。这页重点不是 mAP，而是 dense head、proposal、query、postprocess 如何共同构成端侧感知 workload——尤其是为什么检测的瓶颈常常不在 backbone，而在高分辨率 head 的带宽和 NMS 这类动态后处理的 CPU/NPU 同步。

## 为什么它有效：直觉与类比

检测要同时回答两个问题——"有什么"和"在哪"。难点在于物体数量事先不知道、位置任意，没法像分类那样输出一个固定答案。历史上有两类破解思路，理解它们就理解了整条演进线。

第一类是 **dense prediction**，直觉是**在图像的每个网格点都放一个哨兵，让它就地汇报"我脚下有没有物体、是什么、框该多大"**。一张 feature map 有几千个位置，就有几千个哨兵同时押注，最后把押中的留下。这为什么可行：卷积在每个位置共享同一套权重（见 [CNN Backbone](../01-foundation-model-components/cnn-backbone.md) 的"印章"直觉），所以"训练一个会认物体的哨兵"等于一次性训好了所有位置的哨兵。anchor 则是给每个哨兵预设几种典型框形状当模板（高瘦的人、扁宽的车），让回归从模板微调而非凭空生成。代价是哨兵之间不通气，同一个物体会被相邻好几个哨兵重复押中，于是需要 NMS 事后把重复框去掉——这个去重是个排序+贪心的动态操作，是检测 workload 里最别扭的一块。

第二类是 **query-based**（DETR），直觉是**不再全网格布哨兵，而是派一组固定数量的侦探（比如 100 个 query），每个侦探用 attention 在全图里搜证据、认领一个物体，最后用匈牙利匹配硬性规定"一个物体只能被一个侦探认领"**。为什么这优雅：匹配从机制上保证了不重复，于是 NMS 被彻底删掉——把那块别扭的动态后处理从 pipeline 里拿走了。代价是引入 Transformer decoder 和 cross-attention（见 [Attention and Transformer](../01-foundation-model-components/attention-and-transformer.md)）。这条线之所以重要，是因为它让检测从"dense head + 手工后处理"变成"一组 query + set prediction"，天然接得进 BEV 和 E2E 的统一 query 框架。

## 计算结构与真实量级

```text
image -> backbone -> neck -> detection head -> boxes/classes/scores -> postprocess
```

Two-stage（Faster R-CNN）先出 proposal 再对 ROI 分类回归，精度强但链路长；one-stage（YOLO/RetinaNet）直接在多尺度 feature map 上 dense prediction，实时性好；anchor-free（FCOS/CenterNet）去掉 anchor 组合但仍是 dense per-location；query-based（DETR）把 dense candidate 收成固定数量 query。

量级要落到具体数字。YOLO/RetinaNet 在 FPN 的 P3/P4/P5 上做 dense head：输入 640×640、stride 8 时 P3 约 80×80=6400 个位置，每个位置都要预测类别和 box，head 的输出元素数随 feature area × (类别数 + 框参数) 增长——这是检测 head 带宽压力的来源，远处小目标还逼着系统保留高分辨率 feature，放大 activation memory。DETR 类把候选收成 100-300 个 object query，head 算力骤降，但 decoder 的 cross-attention 和推理时的 bipartite matching 取而代之。NMS 本身是 box 列表上的 `O(n²)` 量级两两 IoU 比较，FLOPs 不高但访问动态 box list、控制流不规则，常落到 CPU 或专用单元，形成 NPU 等 CPU 的同步点。

常见误解：E2E 系统不显式输出 2D box，所以 detection 不重要。实际上检测提供的对象级监督和 query 表达仍影响 scene representation 的稳定性，只是从"输出"退到"中间监督"。

## 一句话理解

目标检测把图像变成对象级稀疏表示；它的 workload 由 backbone/neck 的 dense feature（compute + 高分辨率带宽）和 head/postprocess 的实时路径（NMS 的动态控制流与 CPU/NPU 同步）共同决定，而非单看 backbone 算力。

## 演进与未来方向（判断）

以下为判断，锚定真实趋势。

主线是**检测正从"独立任务 + 手工后处理"溶解为统一 scene transformer 里的一组 query**。DETR 删掉 NMS，DINO/DETR 系把 query 检测做到 SOTA，BEV/E2E（UniAD 类）进一步把 object query、map query、planning query 挂到同一套 scene token 上。我的判断是独立的 2D 检测 head 会越来越少，set-prediction（无 NMS）成为默认。这对架构师是个好消息：**NMS 这类动态、不规则、常需 CPU 介入的后处理被从推理路径里移除**，pipeline 的 CPU/NPU 同步点减少，确定性提升——对追求 deterministic 的 NPU 尤其友好。代价是要支持 query decoder 的 cross-attention 和推理时的匹配，但后者计算量很小。

对 archax 的含义：检测的两种范式落在不同工作点——dense head 是 compute + 高分辨率带宽，且拖着一个 CPU 侧的 NMS 同步尾巴；query head 是 attention（见 [Attention Variants](../01-foundation-model-components/attention-variants-and-efficiency.md) 的 cross-attention 不对称分析）且无后处理尾巴。评估端侧检测时，"有没有 NMS 同步点"应作为 Interaction/Topology 轴上的显式开关，因为它直接决定 p99 latency 的确定性，这一点 06 的端侧芯片需求篇会接着展开。

## Workload Characterization

计算密度：backbone 和 1×1/3×3 dense head 在 channel 足够时 compute-bound；NMS、top-k、box decode 等 postprocess FLOPs 低但 latency 敏感、控制流不规则。

访存模式：多尺度 feature map 读写频繁且规则，高分辨率 P3 是带宽大头；NMS/top-k 访问动态长度的 box list，访问不规则，常触发 CPU/NPU 同步；query decoder 是 cross-attention 的不对称访问（少 query 读大 KV）。

并行性：image batch、feature level、spatial tile、anchor/query 可并行；NMS 和 bipartite matching 有顺序/动态控制，是并行性的断点。

数据复用：backbone/neck feature 被多任务 head 复用（检测/分割/BEV 共享前端）；dense head weight 在空间维复用。

量化敏感度：conv/head 适合 INT8；box decode、score threshold、NMS 通常在更高精度或 CPU/专用后处理执行。

瓶颈类型：端侧常见瓶颈是高分辨率 feature 带宽、head latency、postprocess 同步——dense 检测的 p99 常被 NMS 同步拖累，query 检测则去掉了这条尾巴。

对硬件的核心需求：高效 CNN/Transformer 前端、多尺度 feature buffer、低开销 top-k/NMS（或干脆用 NMS-free query 路线规避）、query decoder 支持、CPU/NPU 同步最小化——详见 [CNN Workload](../06-chip-workload-analysis/cnn-workload.md)。

## 参考来源

- Ren et al., `Faster R-CNN: Towards Real-Time Object Detection with Region Proposal Networks`, NeurIPS 2015, arXiv:1506.01497。成熟度：已落地，two-stage 代表。
- Redmon et al., `You Only Look Once: Unified, Real-Time Object Detection (YOLO)`, CVPR 2016, arXiv:1506.02640。成熟度：已落地，one-stage 实时检测。
- Tian et al., `FCOS: Fully Convolutional One-Stage Object Detection`, ICCV 2019, arXiv:1904.01355。成熟度：已落地，anchor-free 代表。
- Carion et al., `End-to-End Object Detection with Transformers (DETR)`, ECCV 2020, arXiv:2005.12872。成熟度：已落地，query/set-prediction、NMS-free 出处。
