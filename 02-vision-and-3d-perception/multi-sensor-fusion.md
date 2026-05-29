# Multi-sensor Fusion

上级：[Vision and 3D Perception](README.md)
相关：[BEV Perception](bev-perception.md), [LiDAR Point Cloud Processing](lidar-point-cloud-processing.md), [BEV Workload](../06-chip-workload-analysis/bev-workload.md)

## 这页在回答什么问题

多传感器融合回答如何把 camera、LiDAR、radar、IMU 等不同模态对齐到同一个 scene representation。它的 workload 本质是多路 encoder、坐标对齐、特征投影和跨模态融合。

## 计算结构

典型链路是：

```text
camera / lidar / radar
   ->
modality-specific encoders
   ->
projection / calibration / alignment
   ->
fusion in image, point, BEV, token, or object space
   ->
joint scene representation
```

Early fusion 在原始数据附近融合，信息交互充分但标定和对齐复杂，因为不同传感器的采样频率、坐标系、分辨率和噪声模型都不同。Late fusion 在结果级合并，工程简单但互补信息利用不足，因为每个模态已经丢掉了部分中间不确定性。Mid-level fusion 在 feature space 结合，是自动驾驶最常见路径，因为它能在语义和几何之间折中，但代价是中间 feature 搬运和 alignment 成本高。BEV-space fusion 把不同模态投到统一鸟瞰坐标系，是 2022 之后自动驾驶感知的重要主线，因为它把融合坐标系和规划坐标系统一起来。

## 为什么 BEV 成为融合主坐标系

相机是语义强但深度间接，LiDAR 是几何强但语义稀疏，radar 是速度和恶劣天气有优势但分辨率有限。BEV 提供与规划坐标一致的统一空间，把多模态信息变成同一坐标系下的 feature map 或 tokens。

代价是对齐和投影变成关键 workload。camera feature 需要 camera-to-BEV transform，LiDAR feature 需要 voxel/pillar/BEV encoder，radar feature 需要稀疏点或速度特征融合。不同模态频率、延迟和视野不同，还会带来 temporal alignment。一个现实后果是：即使各模态 encoder 都能并行跑，fusion stage 仍然必须等待时间对齐后的 feature，因此 p99 latency 常由最慢模态、最大 feature 搬运或 CPU/NPU 同步点决定。

常见误解：融合就是 concat。实际上，真正困难的是坐标系、时间戳、分辨率、置信度和缺失模态的对齐；concat 只是融合算子的一种表面形式。

## 一句话理解

多传感器融合把多路异构输入变成统一 scene representation；它的核心 workload 是多路并行 encoder 加投影、对齐、跨模态特征搬运和融合。

## Workload Characterization

- 计算密度：各模态 encoder 可能 compute-bound；alignment/projection/concat/attention fusion 常 memory 或 irregular-access-bound。
- 访存模式：camera feature 规则，LiDAR/radar 稀疏；跨模态投影带来 gather/scatter 和 layout conversion。
- 并行性：不同传感器 encoder 可并行；融合 stage 需要同步，受最慢模态和时间对齐约束。
- 数据复用：单模态 feature 可在多任务 head 复用；BEV feature 可作为共享 scene cache。
- 量化敏感度：encoder 可低比特；几何投影、calibration、depth/coordinate 相关部分需要谨慎。
- 瓶颈类型：常见瓶颈是 bandwidth、synchronization、irregular access 和 p99 latency；fusion stage 常是多路 pipeline 的汇合点。
- 对硬件的核心需求：多路 DMA、时间同步 buffer、scatter/gather、BEV feature cache、跨模态 fusion kernel。

## 参考来源

- Liang et al., `Deep Continuous Fusion for Multi-Sensor 3D Object Detection`, ECCV 2018。
- Liu et al., `BEVFusion: Multi-Task Multi-Sensor Fusion with Unified Bird's-Eye View Representation`, arXiv:2205.13542。
- Liang et al., `BEVFusion: A Simple and Robust LiDAR-Camera Fusion Framework`, arXiv:2205.13790。
