# Modular to BEV Evolution

上级：[Autonomous Driving Algorithms](README.md)
相关：[BEV Perception](../02-vision-and-3d-perception/bev-perception.md), [BEV Workload](../06-chip-workload-analysis/bev-workload.md)

## 这页在回答什么问题

这页回答自动驾驶为什么从感知-预测-规划-控制的模块化 pipeline，走向以 BEV / Occupancy 为中心的统一空间表征。重点是模块接口如何限制信息流，以及 BEV 为什么改变 workload。

## 传统模块化 pipeline

传统自动驾驶系统通常组织成：

```text
perception -> tracking -> prediction -> planning -> control
```

这种结构的优点是接口清晰、可调试、可验证。感知输出 object list、lane、traffic light、free space；预测模块输出他车轨迹；规划模块基于规则和优化生成 ego trajectory。

问题在于接口过窄。感知把原始传感器压缩成 box 或 lane 后，很多不确定性、遮挡线索和上下文信息被丢掉；预测和规划只能消费这些固定结构。如果感知漏检，规划往往没有机会从更底层 feature 中恢复。

## BEV 的转折

BEV 引入的是 ego-centric 世界坐标系下的 scene representation：

```text
multi-camera / lidar
   ->
image or point encoder
   ->
view transform / projection
   ->
BEV feature
   ->
detection / map / occupancy / prediction / planning
```

因为规划本来就在地面坐标和轨迹空间中工作，所以 BEV 把感知表示和规划坐标系对齐。它让检测、map、occupancy、motion prediction 可以共享同一个空间底座。

常见误解：BEV 是一个感知 head。实际上，BEV 是中间表示革命；它改变了多摄像头融合、时序缓存、下游任务接口和芯片数据流。

## 从 BEV 到 Occupancy

BEV 仍然是 2D ground-plane 表示，不完整表达高度、遮挡和 3D occupied/free/unknown 状态。Occupancy 把 BEV 推向 3D world state，能更自然描述异形障碍物和可通行空间。

这一步的 workload 变化更大：BEV 的核心是 `camera feature -> BEV grid`，Occupancy 进一步引入 `BEV/3D feature -> voxel grid`，容量和带宽随 voxel resolution、semantic channels、temporal horizon 放大。

## 一句话理解

从模块化到 BEV 的核心是从固定 object/lane 接口转向统一空间表征；它减少了模块边界损失，但把系统瓶颈从规整 CNN 推向 view transform、scatter/gather、BEV cache 和 3D tensor。

## Workload Characterization

- 计算密度：传统模块中 CNN/point cloud encoder 可 compute-bound；BEV view transform 和 occupancy decode 常 memory/irregular-access-bound。
- 访存模式：模块化 object list 轻但信息窄；BEV feature dense 且需要 camera-to-BEV gather/scatter；temporal BEV/occupancy 需要 state cache。
- 并行性：camera、feature level、BEV cell、task head 可并行；fusion 和 planning stage 有同步点。
- 数据复用：BEV feature 可被 detection/map/occupancy/planning 复用；历史 BEV 可跨帧复用。
- 量化敏感度：encoder/head 可低比特；几何投影、depth、插值和 occupancy boundary 需谨慎。
- 瓶颈类型：BEV 阶段常 irregular-access + bandwidth-bound；Occupancy 进一步 capacity-bound。
- 对硬件的核心需求：多路 camera DMA、projection table、scatter/gather、BEV SRAM/DRAM cache、3D tensor tiling。

## 参考来源

- Philion and Fidler, `Lift, Splat, Shoot`, ECCV 2020, arXiv:2008.05711。
- Li et al., `BEVFormer`, ECCV 2022, arXiv:2203.17270。
- Wei et al., `SurroundOcc`, ICCV 2023, arXiv:2303.09551。

## 旧版素材

- `/mnt/e/workload-wiki-old/03_自动驾驶算法路线/从传统模块化到BEV.md`
