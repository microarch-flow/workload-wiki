# Occupancy World Model

上级：[World Model and Generative Intelligence](README.md)
相关：[Occupancy Prediction](../02-vision-and-3d-perception/occupancy-prediction.md), [BEV World Model](bev-world-model.md), [Occupancy Workload](../06-chip-workload-analysis/occupancy-workload.md), [World Model Workload](../06-chip-workload-analysis/world-model-workload.md)

## 这页在回答什么问题

这页回答 occupancy World Model 为什么最贴近安全约束、又为什么是本章 memory 压力最大的一支。它预测未来 3D 空间的 occupied/free/unknown，planner 可直接查询"某个 future pose 是否被占用"；但 4D（3D 空间 × 时间）的容量爆炸把瓶颈从计算推向访存。重点是讲清 voxel 容量如何随 horizon 线性爆炸、为什么 dense 不等于安全、以及离散 occupancy token 自回归这种主流做法的 workload，而非 occupancy 感知的网络结构（在 02 章）。

## 为什么它有效：直觉与类比

occupancy World Model 的直觉是**把世界想成一个会随时间变化的乐高积木盒，每个格子要么有积木、要么空、要么没看清**。planner 要做的安全判断本质很简单——我下一步要去的那几个格子，未来那一刻是不是空的。occupancy 直接把世界离散成 3D voxel grid，每格标 occupied/free/unknown，World Model 预测这个 grid 如何随时间和 ego action 演化。对应到机制，安全查询退化成对 future occupancy grid 的一次 lookup，这是它比 BEV、比 latent 都更直接服务 collision check 的根本原因。

为什么它比 BEV 更适合复杂 3D：BEV 是俯视 2D，悬空物体、坡道、遮挡下的可通行空间表达不全；occupancy 显式建模高度维，悬臂、限高、地形起伏都能落到具体 voxel。对应到机制，多出来的 Z 维让表示在几何上完整，但这正是容量代价的来源。

代价藏在维度里：3D × 时间 = 4D，容量随每一维相乘。对应到机制，这让 occupancy World Model 的瓶颈天然偏向 memory capacity 与 bandwidth，而非 compute——这是它区别于本章其他表示的硬件性格。

## 容量的乘法：为什么访存先撑爆

一个看似不大的 grid 就能说明问题。`200 × 200 × 16` 的 occupancy grid 已是 64 万 cell；再乘 semantic channel（十几类）、future horizon（OccWorld 用 2 秒历史推 3 秒未来，nuScenes 上 2Hz 即推 6 帧）、candidate 数和 batch，中间状态容量迅速变成主问题。横向对比：latent state 是几百维小张量，video latent 大但不查询，occupancy 既大又要被 planner 反复 query——它把 World Model 乘法里的 per-step 成本压在了访存而非计算上。

常见误解：occupancy 越 dense 越安全。实际上 dense grid 只是提供了简单接口，安全来自预测正确而非分辨率高；而 dense 的 memory footprint 随分辨率立方增长，端侧根本装不下。工程上的真实做法是 sparse voxel、multi-resolution、tri-plane 或 region-of-interest——只在有物体/近场/ego 路径附近精细，远处和空旷区粗化。代价是 sparse 带来 irregular access 与 rulebook/neighbor query 的负载不均，瓶颈从 capacity 转成 irregular-access。

## 主流做法：离散 occupancy token 的自回归 rollout

OccWorld（ECCV 2024，arXiv:2311.16038）是这一支的代表，做法很说明 workload：先训一个 3D occupancy scene tokenizer 把 voxel 场景压成离散 high-level token，再用 GPT-like 时空 transformer 在 token 序列上自回归预测下一帧场景（2 秒历史 → 3 秒未来），并联合预测 ego 运动。对应到机制，这把 dense voxel 的容量问题前移到 tokenizer——下游 dynamics 跑在压缩 token 上而非原始 voxel，rollout 成本随 token 数而非 cell 数。但它继承了自回归的硬件性格：逐帧串行、KV cache 随 horizon 增长，是 stateful decode 而非一次 forward。这与 [Diffusion 的步间依赖](diffusion-for-world-model.md)、[Mamba 的状态递推](../01-foundation-model-components/mamba-and-ssm.md) 同属"带状态的迭代推理"主线。

## 一句话理解

occupancy World Model 用 3D voxel 占用预测未来风险，最贴近安全约束（planner 直接查 future pose 是否占用），但 4D 容量爆炸让它成为本章 memory capacity/bandwidth 压力最大的表示；dense 不等于安全，工程上靠 sparse/multi-resolution 与离散 token 自回归把容量压回可行范围。

## 演进与未来方向（判断）

以下为判断，锚定 2024-2026 真实工作。查证日期：2026-06-07。

第一，**occupancy World Model 正从 dense voxel 走向"离散 token + 自回归/扩散"以驯服容量**，并向多模态扩展。OccWorld 用离散 occupancy token + GPT-like 自回归确立了压缩范式；GEM（CVPR 2025，arXiv:2412.11198）这类把 ego-vision 多模态（RGB + depth 等）与细粒度 ego-motion / object dynamics 控制结合，扩展可控性。我的判断是端侧 occupancy World Model 会坚定走 sparse + 压缩 token 路线——dense 4D 在端侧 memory 上不可行，token 化把瓶颈从 capacity 部分转回 compute，更可调度。

第二，落到硬件上：**occupancy World Model 的核心需求是 3D/4D tensor 的 tiling 与 sparse metadata 处理，加上快速 collision query**，这和本章其他表示都不同——它不是 GEMM-bound 也不是 latency-bound，而常是 memory-capacity/bandwidth-bound（dense）或 irregular-access-bound（sparse）。对 archax，voxel 分辨率、semantic channel 数、horizon、sparse 比例是必须显式扫描的维度，sparse conv 的 rulebook、neighbor query、scatter/gather 是相对 dense GEMM 的特殊算子需求；future horizon 把中间状态线性放大，是端侧容量的硬约束。这直接衔接 06 [Occupancy Workload](../06-chip-workload-analysis/occupancy-workload.md)（容量与 sparse 访存）与 [World Model Workload](../06-chip-workload-analysis/world-model-workload.md)（rollout 乘法）。

## Workload Characterization

计算密度：3D conv / sparse conv / voxel transformer / implicit decoder 或离散 token transformer 构成主计算；dense 3D head 的 FLOPs 与 activation 都大，token 化后计算随 token 数而非 cell 数。

访存模式：dense voxel grid 容量大（200×200×16 已 64 万 cell，再乘 channel × horizon × candidate）；sparse 表示带 irregular access；future horizon 线性放大中间状态——访存是本表示的主压力。

并行性：voxel/cell 级并行强；sparse rulebook、neighbor query、temporal update 有同步与负载不均；自回归 rollout 逐帧串行是并行断点。

数据复用：history occupancy、map prior、BEV feature 跨 future step 与 candidate 复用；离散 token 序列可复用 KV cache。

量化敏感度：多数 feature 可量化；occupied/free 边界、小障碍、collision margin 对误差敏感，且误差沿 rollout 累积、直接影响安全查询。

瓶颈类型：dense occupancy 常 memory-capacity/bandwidth-bound；sparse occupancy 常 irregular-access-bound；token 自回归还叠加 KV cache 增长的 stateful-decode 性格。

对硬件的核心需求：3D/4D tensor tiling、sparse metadata（rulebook/neighbor query）处理、voxel cache、future horizon 并发、快速 collision query——详见 [Occupancy Workload](../06-chip-workload-analysis/occupancy-workload.md) 与 [World Model Workload](../06-chip-workload-analysis/world-model-workload.md)。

## 参考来源

- Zheng et al., `OccWorld: Learning a 3D Occupancy World Model for Autonomous Driving`, ECCV 2024 / arXiv:2311.16038。成熟度：occupancy world model 代表，离散 token + GPT-like 自回归，2 秒历史推 3 秒未来，查证日期：2026-06-07。
- Wei et al., `SurroundOcc: Multi-Camera 3D Occupancy Prediction for Autonomous Driving`, ICCV 2023 / arXiv:2303.09551。成熟度：已落地，occupancy 感知基础。
- `OpenOccupancy: A Large Scale Benchmark for Surrounding Semantic Occupancy Perception`, ICCV 2023 / arXiv:2303.03991。成熟度：常用数据/评测资源。
- Hassan et al., `GEM: A Generalizable Ego-Vision Multimodal World Model for Fine-Grained Ego-Motion, Object Dynamics, and Scene Composition Control`, CVPR 2025 / arXiv:2412.11198。成熟度：研究阶段，多模态 ego-vision world model，查证日期：2026-06-07。
</content>
