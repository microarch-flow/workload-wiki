# BEV Perception

上级：[Vision and 3D Perception](README.md)
相关：[Multi-sensor Fusion](multi-sensor-fusion.md), [Autonomous Driving Algorithms](../03-autonomous-driving-algorithms/README.md), [BEV Workload](../06-chip-workload-analysis/bev-workload.md)

## 这页在回答什么问题

BEV Perception 回答为什么自动驾驶把多摄像头、多模态感知投到 Bird's-Eye View 坐标系，以及 camera-to-BEV、temporal BEV 和 BEV fusion 如何改变 workload。它是 02 到 03 自动驾驶算法路线的桥梁。

## BEV 的计算结构

BEV 把前视/环视图像、LiDAR、radar 等输入转成车体坐标系下的 top-down scene representation。

```text
multi-camera images / lidar / radar
   ->
image / point encoders
   ->
view transform / projection / splat / attention query
   ->
BEV feature grid or BEV tokens
   ->
detection / map / occupancy / planning
```

核心难点是 camera image plane 到 BEV plane 的变换。Lift-Splat-Shoot 先预测 depth distribution，把 image feature lift 到 frustum，再 splat 到 BEV grid。BEVFormer 用 BEV queries 通过 spatial cross-attention 从 multi-camera feature 中读取信息，并用 temporal self-attention 融合历史 BEV。BEVFusion 则把 camera/LiDAR 等模态统一到 BEV 空间融合。

典型量级上，LSS 类方法常见 `200 x 200` 量级 BEV grid；多相机输入可能是 6 到 8 路 camera；如果使用 80 个 depth bins 和 `H/16 x W/16` image feature，lift 阶段会形成 `camera x depth x image_feature` 的大中间体。即使最终 BEV grid 只有几十万 cell，camera-to-BEV 的中间 feature、index table 和 pooling/splatting 也会让数据搬运主导 stage。

## 为什么 BEV 对 AD 重要

自动驾驶规划天然在地面坐标系中工作。BEV 把相机视角中的透视畸变、多相机重叠和模态差异统一到规划友好的坐标系。检测、地图、轨迹、occupancy、planning 都可以共享 BEV scene representation。

BEV 也是 03 自动驾驶算法路线的转折点。模块化时代的输出是 object list、lane line、free space 等任务结果；BEV 时代把这些结果前移为共享 scene representation；planning-oriented E2E 进一步把 object query、map query、motion query、planning query 都挂到 BEV 或 scene tokens 上。UniAD 类系统的关键不是“多加几个 head”，而是让 perception、prediction、planning 通过共享表示和 query 机制形成统一计算图。

常见误解：BEV 只是把图像换个视角。实际上，BEV 是一个系统级表示选择；它决定了多传感器融合、时序缓存、下游任务接口和硬件数据流。

## BEV 的主要路线

| 路线 | 机制 | Workload 影响 |
| --- | --- | --- |
| Geometry projection | 利用相机内外参和 depth 把 feature 投到 BEV | depth bins、splat、scatter/gather 重 |
| Query-based BEV | BEV/object query 从 image features 中 cross-attention 读取 | attention + sampling，query/KV 不对称，适合接 planning query |
| LiDAR/Pillar BEV | 点云 pillar/voxel 投到 BEV | sparse -> dense BEV，metadata 与投影开销 |
| BEV Fusion | 多模态 feature 在 BEV 空间融合 | 多路 encoder 同步，BEV feature cache |
| Temporal BEV | 历史 BEV 与当前 BEV 融合 | state cache、ego-motion alignment、latency |

## 一句话理解

BEV 是自动驾驶从感知走向规划的统一坐标系；它的 workload 核心不是单个卷积，而是多相机/多模态到 BEV 的投影、采样、scatter/gather 和 temporal cache。

## Workload Characterization

- 计算密度：image encoder 和 BEV encoder 可 compute-bound；view transform、splat、sampling、ego-motion alignment 常 memory/irregular-access-bound。
- 访存模式：multi-camera feature 规则但体量大；camera-to-BEV 映射是 gather/scatter；temporal BEV cache 是 stateful access。
- 并行性：camera、BEV cell、depth bin、query、history frame 可并行；fusion 和 temporal alignment 有同步点。
- 数据复用：image feature、depth distribution、BEV cache、calibration table 可复用；历史 BEV 可跨帧复用。
- 量化敏感度：CNN/BEV encoder 可 INT8；depth distribution、坐标投影、插值采样、BEV alignment 需要谨慎。
- 瓶颈类型：camera-to-BEV 常 irregular-access + bandwidth-bound；temporal BEV 可能 capacity/latency-bound。
- 对硬件的核心需求：多路 camera DMA、scatter/gather、projection table、BEV SRAM/DRAM cache、temporal state update、高效 BEV encoder，以及 query 到 BEV feature 的低延迟读取。

## 参考来源

- Philion and Fidler, `Lift, Splat, Shoot: Encoding Images from Arbitrary Camera Rigs by Implicitly Unprojecting to 3D`, ECCV 2020, arXiv:2008.05711。
- Li et al., `BEVFormer: Learning Bird's-Eye-View Representation from Multi-Camera Images via Spatiotemporal Transformers`, ECCV 2022, arXiv:2203.17270。
- Huang et al., `BEVDet: High-performance Multi-camera 3D Object Detection in Bird-Eye-View`, arXiv:2112.11790。
- Li et al., `BEVDepth: Acquisition of Reliable Depth for Multi-view 3D Object Detection`, arXiv:2206.10092。
- Liu et al., `PETR: Position Embedding Transformation for Multi-View 3D Object Detection`, ECCV 2022, arXiv:2203.05625。
- Liu et al., `BEVFusion: Multi-Task Multi-Sensor Fusion with Unified Bird's-Eye View Representation`, arXiv:2205.13542。

## 旧版素材

- 新增页面，旧版 02 中没有独立 BEV 文档；素材来自旧版多传感器融合、Occupancy 和 03 自动驾驶 BEV 相关章节。
