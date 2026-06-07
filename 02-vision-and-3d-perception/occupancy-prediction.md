# Occupancy Prediction

上级：[Vision and 3D Perception](README.md)
相关：[BEV Perception](bev-perception.md), [Occupancy Workload](../06-chip-workload-analysis/occupancy-workload.md), [World Model Workload](../06-chip-workload-analysis/world-model-workload.md)

## 这页在回答什么问题

Occupancy Prediction 回答三维空间中哪里被占据、哪里空闲、哪里未知，必要时还预测语义类别。它把感知从对象和像素推进到 3D world state，是自动驾驶和机器人走向 World Model 的关键表示。这页重点是它的 workload 签名——3D/4D 张量的容量爆炸，以及表示选择（dense/sparse/tri-plane/implicit）如何直接换不同的硬件瓶颈。

## 为什么它有效：直觉与类比

occupancy 的直觉是**给整个 3D 空间做一次体素 CT 扫描**。检测是在地图上插旗标出"这里有辆车、那里有个人"——只标得出认识的离散对象；occupancy 是把车周围空间切成密密麻麻的小立方体（voxel），逐个回答"这一格是被占据、是空的、还是未知"。为什么这比检测更适合规划：真实道路上有大量检测框不出来的东西——侧翻的卡车、掉落的轮胎、施工的异形围挡、半遮挡的怪异障碍。检测认不出就漏报，occupancy 不管那是什么，只要那一格被占了就标占据，于是异形障碍、可通行空间、未知区域都被统一表达成空间状态——这正是 planning 和 collision checking 真正需要的信息。

最该注意的是"未知"这一类，它是 occupancy 区别于 3D 语义分割的灵魂。被遮挡的、激光没扫到的格子不是"空"，而是"不知道"——而规划对"空"和"不知道"的态度完全不同（空可以开过去，不知道要保守）。所以 occupancy 的目标不是给每个 voxel 分类，而是为决策提供带不确定性的世界状态。常见误解：Occupancy 是 3D 语义分割。实际上它必须表达 free/unknown/occupied 三态、且服务于规划与世界状态，而非单纯逐 voxel 贴语义标签。

表示选择上有几种省容量的巧思，各有直觉。**dense voxel** 是老老实实给每格存一个值，规整但容量爆炸。**tri-plane** 是**把 3D 拆成三张正交的 2D 投影照**（俯视、侧视、正视），用三张 2D 平面的组合隐式表达 3D，容量从立方降到平方，代价是查一个 3D 点要跨三个平面采样再合成。**implicit/NeRF-like** 是**不存网格、存一个函数**，要哪个点的占据就用 query 现场算，连续且省存储，但推理时 query 点数和采样变成不规则访问。

## 计算结构与容量爆炸的真实账

```text
multi-view images / lidar / fused BEV -> BEV or 3D representation
   -> voxel / tri-plane / occupancy decoder -> occupied / free / semantic occupancy
```

camera-only occupancy 靠深度和多视角几何推断，multi-modal 用 LiDAR 几何补强（见 [Multi-sensor Fusion](multi-sensor-fusion.md)）。

把容量账算清楚，这是 occupancy 的核心矛盾：用 `200×200×16` voxel grid 就有 64 万个 cell；每个 cell 输出 18 个语义 logits，单帧 logits 就超过 1100 万个值。这还只是单帧——一旦引入 4-8 帧 temporal occupancy 或预测 future occupancy，容量和带宽按 horizon 线性放大，迅速变成 memory-capacity-bound。这就是为什么 occupancy 的 workload 性格和前面所有 2D 任务都不同：2D 分割的痛是高分辨率输出的带宽，occupancy 的痛是 3D/4D 张量的绝对容量——它能不能端侧跑，常常先卡在 activation buffer 装不装得下，而不是算力够不够。

表示选择就是在换瓶颈：dense voxel 访问规则但容量爆炸（capacity-bound）；sparse voxel 只算非空/高置信区域省容量，但引入 metadata、hash/rulebook、负载不均（irregular-access-bound，同 [LiDAR](lidar-point-cloud-processing.md) 的稀疏代价）；tri-plane 省容量但要跨平面 gather；implicit 省存储但 query 采样不规则。没有免费的午餐，只有"把容量压力换成访存不规则压力"的不同配比。

## 为什么它是通向 World Model 的桥

在 World Model 里，future occupancy prediction 是最自然的落地形式之一：不必生成逼真视频，只要预测未来的空间占据和风险状态即可（这正是 [JEPA](../01-foundation-model-components/jepa-and-self-supervised.md) "预测要点不预测像素"思想的 3D 落地）。OccWorld 这类路线已把 occupancy token 当 world state，预测未来 occupancy 和 ego trajectory——occupancy 因此是 02 感知通向 05/06 World Model 的关键桥梁。

## 一句话理解

Occupancy 把场景表示从对象级、图像级推进到带"未知"态的 3D 空间状态，为规划提供检测给不了的异形/可通行/未知信息；它的 workload 签名是 3D/4D 张量的容量爆炸，表示选择（dense/sparse/tri-plane/implicit）本质是在 capacity-bound 与 irregular-access-bound 之间换瓶颈。

## 演进与未来方向（判断）

以下为判断，锚定真实趋势。

第一，**occupancy 正在成为自动驾驶 3D 场景的标准表示，并和 BEV、World Model 三者合流**。BEV 给俯视特征，occupancy 给 3D 状态，二者底层都建在统一 voxel/BEV 空间上（见 [BEV Perception](bev-perception.md)）；再加上 future occupancy = 轻量 world model，三条线正收敛成"统一 3D 场景表示 + 时序预测"。我的判断是 occupancy 会取代纯检测成为量产感知的核心输出之一——特斯拉已公开把 occupancy network 作为主感知，理由就是它对异形障碍和可通行空间的覆盖是检测给不了的。

第二条对架构师最关键：**occupancy 的 3D/4D 容量是端侧最硬的约束，且 future occupancy 把它从"一次推理"变成"带状态的时序 rollout"**。单帧就 1100 万 logits，加 horizon 后线性膨胀——这不是靠提算力能解决的，得靠表示压缩（sparse/tri-plane）和时序状态复用。对 archax，occupancy 应建模为容量主导的工作点，**核心可建模参数是 grid 分辨率 × 语义类别数 × temporal horizon**，三者任一上调都线性放大 capacity/bandwidth 压力；表示选择（dense vs sparse）是 Capability/Interaction 轴上的关键开关，决定落在 capacity-bound 还是 irregular-access-bound。future occupancy 的 rollout 步则和 [Mamba](../01-foundation-model-components/mamba-and-ssm.md)、[Diffusion](../01-foundation-model-components/diffusion-models.md) 一样是带状态的迭代维度。这整套正是 06 的 [Occupancy Workload](../06-chip-workload-analysis/occupancy-workload.md) 和 [World Model Workload](../06-chip-workload-analysis/world-model-workload.md) 的核心。

## Workload Characterization

计算密度：dense 3D decoder 和 voxel attention 计算重；sparse occupancy 降低计算但引入 metadata/索引开销，有效算术强度取决于稀疏调度。

访存模式：dense voxel 规则但容量巨大；sparse voxel 需 metadata/rulebook 的不规则访问；tri-plane 需跨 plane gather；implicit query 不规则；temporal occupancy 需状态缓存。

并行性：voxel/cell、semantic channel、camera、time horizon 可并行；稀疏分布造成负载不均。

数据复用：BEV feature、depth feature、temporal state 可复用；3D decoder 中间 feature 占用大，是容量压力源。

量化敏感度：occupancy logits 可低比特；几何坐标、depth、free/unknown 边界、sparse metadata 需谨慎（边界与不确定性对误差敏感）。

瓶颈类型：dense occupancy 常 memory-capacity-bound（这是它区别于 2D 任务的签名）；sparse occupancy 常 irregular-access-bound；future occupancy 可能 rollout-latency-bound。

对硬件的核心需求：大容量 activation buffer、3D/4D tensor tiling、sparse metadata/rulebook、BEV-to-voxel decode、tri-plane sampling、低延迟 temporal cache——容量是第一约束，详见 [Occupancy Workload](../06-chip-workload-analysis/occupancy-workload.md)。

## 参考来源

- Wei et al., `SurroundOcc: Multi-Camera 3D Occupancy Prediction for Autonomous Driving`, ICCV 2023, arXiv:2303.09551。成熟度：已落地研究，多相机 occupancy 代表。
- Huang et al., `Tri-Perspective View for Vision-Based 3D Semantic Occupancy Prediction (TPVFormer)`, CVPR 2023, arXiv:2302.07817。成熟度：已落地研究，tri-plane 表示代表。
- Tian et al., `Occ3D: A Large-Scale 3D Occupancy Prediction Benchmark for Autonomous Driving`, NeurIPS 2023 D&B, arXiv:2304.14365。成熟度：基准数据集。
- Zheng et al., `OccWorld: Learning a 3D Occupancy World Model for Autonomous Driving`, ECCV 2024, arXiv:2311.16038。成熟度：研究阶段，occupancy world model 代表，通向 05/06。
