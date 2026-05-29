# Vision and 3D Perception

上级：[Workload Wiki](../README.md)
相关：[Foundation Model Components](../01-foundation-model-components/README.md), [Autonomous Driving Algorithms](../03-autonomous-driving-algorithms/README.md), [Chip Workload Analysis](../06-chip-workload-analysis/README.md)

## 这章在回答什么问题

这一章回答视觉和 3D 感知如何把原始传感器输入变成对象、区域、点云、BEV、Occupancy 和视频状态表示。它是 03 自动驾驶算法、04 机器人 VLA、05 World Model 的感知输入层，也是 06 workload 分析中 BEV、Occupancy、E2E 的直接素材。

## 本章定位

02 不写成普通 CV 任务综述，而是围绕“表示形态如何改变 workload”组织。2D detection/segmentation 是对象和区域表示，LiDAR 是稀疏 3D 表示，多传感器融合和 BEV 是统一坐标系表示，Occupancy 是 3D world state，Video Understanding 是时序状态表示。

## 页面列表

- [Object Detection](object-detection.md)
- [Semantic Segmentation](semantic-segmentation.md)
- [Instance Segmentation](instance-segmentation.md)
- [LiDAR Point Cloud Processing](lidar-point-cloud-processing.md)
- [Multi-sensor Fusion](multi-sensor-fusion.md)
- [Occupancy Prediction](occupancy-prediction.md)
- [Video Understanding](video-understanding.md)
- [BEV Perception](bev-perception.md)

## 到 06 的映射

| 感知表示 | 主要 workload | 06 对应入口 |
| --- | --- | --- |
| 2D detection / segmentation | backbone、neck、dense head、postprocess | [CNN Workload](../06-chip-workload-analysis/cnn-workload.md) |
| LiDAR point cloud | voxelization、sparse conv、pillar/BEV projection | [Occupancy Workload](../06-chip-workload-analysis/occupancy-workload.md) |
| Multi-sensor fusion | modality encoder、projection、alignment、fusion | [BEV Workload](../06-chip-workload-analysis/bev-workload.md) |
| BEV perception | camera-to-BEV、temporal BEV、BEV encoder | [BEV Workload](../06-chip-workload-analysis/bev-workload.md) |
| Occupancy | 3D/4D voxel、semantic occupancy、future occupancy | [Occupancy Workload](../06-chip-workload-analysis/occupancy-workload.md) |
| Video understanding | temporal encoder、history cache、video tokens | [E2E Workload](../06-chip-workload-analysis/e2e-workload.md), [World Model Workload](../06-chip-workload-analysis/world-model-workload.md) |
