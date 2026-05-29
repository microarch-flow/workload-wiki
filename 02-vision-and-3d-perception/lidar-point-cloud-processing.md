# LiDAR Point Cloud Processing

上级：[Vision and 3D Perception](README.md)
相关：[Multi-sensor Fusion](multi-sensor-fusion.md), [Occupancy Prediction](occupancy-prediction.md), [Occupancy Workload](../06-chip-workload-analysis/occupancy-workload.md)

## 这页在回答什么问题

LiDAR 点云处理回答如何从稀疏、不规则 3D 点集合中提取几何和对象信息。它的核心 workload 挑战不是 3D 本身，而是不规则稀疏数据如何映射到规整 NPU。

## 计算结构

点云输入是一组 `(x, y, z, intensity, timestamp)` 点，不是 dense image grid。典型链路是：

```text
raw points -> voxelization / pillarization -> sparse or BEV features -> 3D detection / segmentation / occupancy
```

Point-based 方法直接在点上做邻域搜索和聚合，几何细节保留好但硬件不规则。Voxel-based 方法把点量化到 3D grid，再用 sparse conv 或 3D conv。Sparse conv 不是简单跳过空 voxel，它需要为每个卷积层构建 active voxel 的邻接关系，常见实现会维护 rulebook / indice pairs，用来描述哪些输入 voxel 参与哪些输出 voxel。Pillar/BEV 方法把高度维压缩成柱状表示，换取更高端侧效率。

典型量级上，自动驾驶单帧 LiDAR 可以有数万到十几万点；voxel/pillar 后有效 cell 远少于 dense 3D grid，但需要 metadata、索引和 scatter/gather。以 `X x Y x Z = 400 x 400 x 16` 的 dense grid 为例，理论 cell 超过 250 万，但真实 active voxel 可能只占很小比例；省下的计算会被 rulebook 构建、indice gather 和负载不均部分抵消。

## 表示演进

点云处理从 hand-crafted feature 走向 PointNet、voxel sparse conv、PointPillars/BEV。演进方向很清楚：越保留原始点，表达越直接但越不规则；越投到 voxel/pillar/BEV，硬件越规整但可能损失细节。

常见误解：稀疏就一定省。实际上，稀疏表示省掉了无效 voxel 的计算，但引入索引、metadata、负载不均和 gather/scatter；如果硬件不支持稀疏调度，稀疏 workload 可能并不快。

## 一句话理解

LiDAR workload 的核心是稀疏 3D 到规整表示的转换；它在几何表达上强，但对 voxelization、sparse conv、metadata 和不规则访存提出硬件挑战。

## Workload Characterization

- 计算密度：dense 3D conv 很重；sparse conv 减少无效计算但有效计算密度取决于稀疏调度；pillar/BEV 后更接近 2D conv。
- 访存模式：raw point 和 voxelization 是 scatter/gather；sparse conv 依赖索引、hash/rulebook 和邻接表；BEV feature 访问更规则。
- 并行性：point、voxel、pillar 可并行；稀疏分布不均会导致负载不均。
- 数据复用：dense conv 有局部复用；sparse conv 复用受 occupancy pattern 影响；BEV projection 后复用更好。
- 量化敏感度：BEV/CNN 部分适合 INT8；坐标量化、voxel indexing、几何聚合需要保持精度。
- 瓶颈类型：voxelization 和 sparse conv 常 irregular-access-bound；dense occupancy decoder 可能 capacity-bound。
- 对硬件的核心需求：scatter/gather、sparse metadata、indexed load/store、负载均衡、BEV projection 和 2D/3D DMA。

## 参考来源

- Qi et al., `PointNet: Deep Learning on Point Sets for 3D Classification and Segmentation`, CVPR 2017, arXiv:1612.00593。
- Yan et al., `SECOND: Sparsely Embedded Convolutional Detection`, Sensors 2018, arXiv:1806.05578。
- Lang et al., `PointPillars: Fast Encoders for Object Detection from Point Clouds`, CVPR 2019, arXiv:1812.05784。

## 旧版素材

- `/mnt/e/workload-wiki-old/02_视觉与3D感知/激光雷达点云处理/激光雷达点云处理总览.md`
