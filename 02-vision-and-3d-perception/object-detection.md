# Object Detection

上级：[Vision and 3D Perception](README.md)
相关：[CNN Backbone](../01-foundation-model-components/cnn-backbone.md), [BEV Perception](bev-perception.md), [CNN Workload](../06-chip-workload-analysis/cnn-workload.md)

## 这页在回答什么问题

目标检测回答图像或场景中有哪些对象、在哪里。这里重点不是 mAP 细节，而是理解 dense head、proposal、query 和 postprocess 如何形成端侧感知 workload。

## 计算结构

经典检测链路是：

```text
image -> backbone -> neck -> detection head -> boxes/classes/scores -> postprocess
```

Two-stage 检测先生成 proposal，再对 ROI 做分类和回归，精度强但流程长。One-stage 检测直接在多尺度 feature map 上做 dense prediction，实时性更好。Anchor-free 方法减少 anchor 先验，query-based detection 则把对象表示成一组 learnable queries，更容易接入 Transformer、BEV 和 E2E 系统。

从 workload 看，检测不只是 backbone。多尺度 neck、高分辨率 dense head、box regression、classification、NMS 或 matching/postprocess 都会进入 latency path。自动驾驶中小目标和远距离目标迫使系统保留较高分辨率 feature，因此 bandwidth 和 activation memory 不能忽略。

YOLO/RetinaNet 这类 one-stage dense detector 通常在 FPN 的 P3/P4/P5 上做 dense head。以 P3 为例，如果输入 640x640 且 stride 为 8，P3 feature 约为 80x80；每个位置都要预测类别和 box，因此 head 的输出元素数与 feature area 和类别数强相关。FCOS/CenterNet 类 anchor-free 方法减少 anchor 组合，但仍然是 dense per-location prediction。DETR 类 query detector 把 dense candidate 减少为固定数量 object queries，例如 100-300 个 query，但引入 Transformer decoder 和 bipartite matching。

## 演进与系统意义

检测从滑窗/手工特征演进到 two-stage、one-stage、anchor-free 和 query-based，本质是从密集候选和规则后处理走向更统一的对象表示。自动驾驶早期把 box 当作规划输入；BEV/E2E 时代，检测更多成为对象级 supervision 或稀疏 scene tokens 的一部分。

常见误解：E2E 系统不显式输出 2D box，所以 detection 不重要。实际上，检测提供的对象级监督和 query 表达仍然会影响 scene representation 的稳定性。

## 一句话理解

目标检测把图像变成对象级稀疏表示；它的 workload 由 backbone/neck 的 dense feature 和 head/postprocess 的实时路径共同决定。

## Workload Characterization

- 计算密度：backbone 和 1x1/3x3 head 可 compute-bound；NMS、top-k、decode 等 postprocess FLOPs 低但 latency 敏感。
- 访存模式：多尺度 feature map 读写频繁，dense head 访问规则；NMS/top-k 访问 box list，动态性更强，常导致 CPU/NPU 同步。
- 并行性：image batch、feature level、spatial tile、anchor/query 可并行；NMS 和 matching 有顺序/动态控制。
- 数据复用：backbone/neck feature 可被多任务 head 复用；检测 head weight 在空间维复用。
- 量化敏感度：conv/head 适合 INT8；box decode、score threshold、NMS 通常在更高精度或 CPU/专用后处理中执行。
- 瓶颈类型：端侧常见瓶颈是 high-resolution feature bandwidth、head latency 和 postprocess sync。
- 对硬件的核心需求：高效 CNN/Transformer 前端、多尺度 feature buffer、低开销 top-k/NMS、query decoder 支持、CPU/NPU 同步最小化。

## 参考来源

- Ren et al., `Faster R-CNN`, NeurIPS 2015, arXiv:1506.01497。
- Redmon et al., `You Only Look Once`, CVPR 2016, arXiv:1506.02640。
- Carion et al., `End-to-End Object Detection with Transformers`, ECCV 2020, arXiv:2005.12872。

## 旧版素材

- `/mnt/e/workload-wiki-old/02_视觉与3D感知/目标检测/目标检测总览.md`
