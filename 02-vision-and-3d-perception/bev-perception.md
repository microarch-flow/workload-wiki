# BEV Perception

上级：[Vision and 3D Perception](README.md)
相关：[Multi-sensor Fusion](multi-sensor-fusion.md), [Autonomous Driving Algorithms](../03-autonomous-driving-algorithms/README.md), [BEV Workload](../06-chip-workload-analysis/bev-workload.md)

## 这页在回答什么问题

BEV Perception 回答为什么自动驾驶把多摄像头、多模态感知投到 Bird's-Eye View 坐标系，以及 camera-to-BEV、temporal BEV、BEV fusion 如何改变 workload。它是 02 感知到 03 自动驾驶算法路线的桥梁——理解 BEV 的视角变换为什么是 irregular-access 主导，就理解了 06 [BEV Workload](../06-chip-workload-analysis/bev-workload.md) 的核心难题。

## 为什么它有效：直觉与类比

BEV 的直觉是**把装在车身不同位置、各看一个方向的几个相机拍的透视照片，拼接、矫正、俯视投影成一张上帝视角的地图**。为什么非要这张上帝视角图：因为规划本来就在地面坐标系里做决策——"前方 30 米变道""与左侧车距 2 米"都是俯视坐标系里的量。而原始相机图是透视的，近大远小、多相机视野重叠、同一个物体在不同相机里位置不同，根本没法直接拿去规划。BEV 把透视畸变、多相机重叠、模态差异统一到规划友好的俯视坐标系，让检测、地图、轨迹、occupancy、planning 都能共享同一份 scene representation——这是它的根本价值。

难点全在"透视照片 → 俯视地图"这一步变换上，而这步有两种方向相反的做法，理解这对对照就抓住了 BEV 的 workload 本质。

**Lift-Splat-Shoot（LSS）是 forward scatter**，直觉是**每个像素先猜自己有多远，再按猜测的距离把自己抛洒到地图对应格子上**。相机图每个像素预测一个深度分布（它可能在 5 米、也可能 20 米，给个概率），据此把该像素的特征"lift"到 3D 视锥（frustum），再"splat"抛洒到 BEV 网格。这是 scatter——从像素出发，把特征撒向它可能落到的 BEV 格子。

**BEVFormer 是 backward gather + attention**，直觉正好反过来——**BEV 地图的每个格子主动派一个 query，按相机标定参数算出"我这个格子对应到各相机图的哪个位置"，去那里采样、cross-attention 读回特征**。这是 gather——从 BEV 格子出发，按几何反查去各相机图里取特征，再用 temporal self-attention 融合历史 BEV。

两种做法都不是规整 GEMM，而是由相机几何决定的 scatter（LSS）或 gather（BEVFormer）——访问位置由标定和深度动态决定、不连续、破坏 DRAM row locality。这就是为什么 BEV 的瓶颈 stage 是 memory/irregular-access 主导而非算力主导，下一节量化它。

## 计算结构与那个大中间体

```text
multi-camera images / lidar / radar -> image / point encoders
   -> view transform / projection / splat / attention query -> BEV feature grid or BEV tokens
   -> detection / map / occupancy / planning
```

把量级摆出来。LSS 类常见 `200×200` 量级 BEV grid，多相机输入 6-8 路。关键的成本不在最终 grid（几十万 cell），而在 lift 阶段的中间体：若用 80 个 depth bins、`H/16 × W/16` image feature，会形成 `camera × depth × image_feature` 的巨大 frustum 中间张量——6 相机 × 80 depth × 几千 feature 位置，这个中间体比输入和输出都大得多，且它的 splat/pooling 到 BEV 是 scatter。BEVFormer 侧则是每个 BEV query 通过 deformable attention 在多相机、多尺度上采样少量点（见 [Attention Variants](../01-foundation-model-components/attention-variants-and-efficiency.md) 的 deformable 分析），把成本从 dense frustum 换成动态 gather。无论哪种，camera-to-BEV 的中间 feature、index/projection table、pooling/splatting 让数据搬运主导这个 stage。

常见误解：BEV 只是把图像换个视角。实际上 BEV 是系统级表示选择——它决定多传感器融合方式、时序缓存策略、下游任务接口和整条硬件数据流。

## 为什么 BEV 是 02→03 的转折点

BEV 也是 03 自动驾驶算法路线的转折。模块化时代输出的是 object list、lane line、free space 等任务结果；BEV 时代把这些结果前移为共享 scene representation；planning-oriented E2E（UniAD 类）进一步把 object query、map query、motion query、planning query 都挂到 BEV 或 scene token 上。UniAD 的关键不是"多加几个 head"，而是让 perception、prediction、planning 通过共享表示和 query 机制形成统一计算图——这正是 [03 章](../03-autonomous-driving-algorithms/README.md) 的主线。

主要路线对照：

| 路线 | 机制 | Workload 影响 |
| --- | --- | --- |
| Geometry projection (LSS/BEVDet) | 预测 depth，把 feature lift+splat 到 BEV | frustum 大中间体、depth bins、scatter |
| Query-based (BEVFormer/PETR) | BEV/object query 从 image feature cross-attention 读取 | deformable sampling，query/KV 不对称，易接 planning query |
| LiDAR/Pillar BEV | 点云 pillar/voxel 投到 BEV | sparse→dense BEV，metadata 与投影开销 |
| BEV Fusion | 多模态 feature 在 BEV 空间融合 | 多路 encoder 同步，BEV feature cache（见 multi-sensor-fusion） |
| Temporal BEV | 历史 BEV 与当前 BEV 融合 | state cache、ego-motion alignment、latency |

## 一句话理解

BEV 是自动驾驶从感知走向规划的统一坐标系；它的 workload 核心不是单个卷积，而是 camera-to-BEV 的视角变换——LSS 的 forward scatter 或 BEVFormer 的 backward gather+attention，都是几何驱动的不规则访存，再叠加 temporal BEV 的状态缓存。

## 演进与未来方向（判断）

以下为判断，锚定真实趋势。

第一，**dense BEV grid 正在向 sparse query-centric 表示演进**，以缓解那个大中间体的成本。早期 LSS 维护稠密 frustum 和稠密 BEV grid，新一代（SparseBEV、稀疏 query 路线）只在感兴趣的稀疏 query 上做 deformable 采样，绕开稠密 frustum 的容量和 scatter 开销。我的判断是 query-centric 会成为主流——它不仅省成本，还天然对接 E2E 的统一 query 框架（object/map/planning query 和 BEV query 同源）。对架构师，这意味着 BEV 的瓶颈从"稠密 scatter + 大 frustum 容量"逐步转向"deformable attention 的动态 gather"——两者都是 irregular access，但后者把容量压力换成了采样不规则压力。

第二，**BEV、occupancy、temporal memory 三者正合流成统一的时空 3D 场景表示**（见 [Occupancy Prediction](occupancy-prediction.md)）。BEV 提供俯视特征，occupancy 补上高度维的 3D 状态，temporal BEV 提供历史记忆，再往前就是 World Model 的 action-conditioned future。对 archax，BEV perception 应建模为多个不同性格工作点的串联：image encoder（compute-bound）→ view transform（irregular-access-bound，是签名 stage）→ BEV encoder（compute-bound）→ temporal fusion（stateful、latency 敏感）。其中**view transform 是必须显式建模的 irregular-access 极端点**，不能和前后的规整 encoder 用平均值合并——这正是 06 [BEV Workload](../06-chip-workload-analysis/bev-workload.md) 的核心，也是 BEV 对 deterministic NPU 最大的挑战：scatter/gather、projection table、动态采样这些算子，恰是规整 MAC 阵列最不擅长的。

## Workload Characterization

计算密度：image encoder 和 BEV encoder 可 compute-bound；view transform、splat、sampling、ego-motion alignment 常 memory/irregular-access-bound——这是 BEV 的签名分裂。

访存模式：multi-camera feature 规则但体量大；camera-to-BEV 映射是 gather/scatter（LSS scatter，BEVFormer gather）；temporal BEV cache 是 stateful access；LSS 的 frustum 是大中间体。

并行性：camera、BEV cell、depth bin、query、history frame 可并行；fusion 和 temporal alignment 有同步点（见 [Multi-sensor Fusion](multi-sensor-fusion.md)）。

数据复用：image feature、depth distribution、calibration/projection table、BEV cache 可复用；历史 BEV 跨帧复用是 temporal BEV 的核心。

量化敏感度：CNN/BEV encoder 可 INT8；depth distribution、坐标投影、插值采样、BEV alignment 需谨慎（几何误差直接放大成定位误差）。

瓶颈类型：camera-to-BEV 常 irregular-access + bandwidth-bound；temporal BEV 可能 capacity/latency-bound；整条链 p99 受 view transform 和 temporal 同步影响。

对硬件的核心需求：多路 camera DMA、scatter/gather、projection/index table、BEV SRAM/DRAM cache、temporal state update、高效 BEV encoder，以及 query 到 BEV feature 的低延迟读取——view transform 的不规则访存是对 deterministic NPU 的核心挑战，详见 [BEV Workload](../06-chip-workload-analysis/bev-workload.md)。

## 参考来源

- Philion and Fidler, `Lift, Splat, Shoot: Encoding Images from Arbitrary Camera Rigs by Implicitly Unprojecting to 3D`, ECCV 2020, arXiv:2008.05711。成熟度：已落地，forward scatter（LSS）出处。
- Li et al., `BEVFormer: Learning Bird's-Eye-View Representation from Multi-Camera Images via Spatiotemporal Transformers`, ECCV 2022, arXiv:2203.17270。成熟度：已落地，backward gather + 时空 attention 代表。
- Huang et al., `BEVDet: High-performance Multi-camera 3D Object Detection in Bird-Eye-View`, arXiv:2112.11790。成熟度：已落地，LSS 系工程化。
- Liu et al., `PETR: Position Embedding Transformation for Multi-View 3D Object Detection`, ECCV 2022, arXiv:2203.05625。成熟度：已落地，隐式 query BEV。
- Liu et al., `BEVFusion: Multi-Task Multi-Sensor Fusion with Unified Bird's-Eye View Representation`, ICRA 2023 / arXiv:2205.13542。成熟度：已落地，BEV 多模态融合。
- Liu et al., `SparseBEV: High-Performance Sparse 3D Object Detection from Multi-Camera Videos`, ICCV 2023, arXiv:2308.09244。成熟度：已落地研究，sparse query-centric BEV 代表。
