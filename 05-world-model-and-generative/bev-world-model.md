# BEV World Model

上级：[World Model and Generative Intelligence](README.md)
相关：[BEV Perception](../02-vision-and-3d-perception/bev-perception.md), [BEV Workload](../06-chip-workload-analysis/bev-workload.md)

## 这页在回答什么问题

这页回答 BEV World Model 为什么适合自动驾驶。BEV 已经把多相机和 LiDAR 信息对齐到 ego-centric ground plane，World Model 在 BEV 上预测未来，可以直接服务 motion forecasting、planning 和 risk scoring。

## 基本结构

```text
history sensor frames
   ->
BEV encoder
   ->
BEV temporal dynamics
   ->
future BEV feature / object motion / cost map
   ->
planner or simulator
```

BEV World Model 不追求生成逼真像素，而追求未来空间状态对规划有用。它可以预测 future BEV feature、actor motion、lane occupancy、cost map 或 driving scene latent。

## 为什么是 BEV

自动驾驶规划天然在鸟瞰坐标中工作。相比视频，BEV 更容易表达距离、速度、轨迹和碰撞关系；相比纯 latent，BEV 更可解释，也更容易接入 rule check 和 planner。

问题是 BEV 仍然是 2D 表示，难以完整表达高度、遮挡和立体占用。对复杂 3D 障碍、坡道、悬空物体和遮挡恢复，occupancy world model 更自然。

## 一句话理解

BEV World Model 是自动驾驶 planning 友好的未来预测表示；它用空间结构换取可评估性，但 workload 受 BEV grid、时序缓存和多候选 rollout 共同影响。

## Workload Characterization

- 计算密度：BEV encoder 和 temporal dynamics 是主计算；未来 head 通常比视频 decoder 轻。
- 访存模式：BEV feature map、history BEV cache、ego-motion alignment 和 future horizon 产生持续读写。
- 并行性：BEV cell、future candidate、task head 可并行；temporal rollout 受时间依赖限制。
- 数据复用：同一 BEV feature 可服务 detection、motion、planning、world model rollout。
- 量化敏感度：BEV backbone 可量化；几何对齐、collision boundary、trajectory cost 对误差敏感。
- 瓶颈类型：view transform + BEV cache 是前端瓶颈；future rollout 的瓶颈取决于 horizon 和 candidate 数。
- 对硬件的核心需求：BEV feature 缓存、时序对齐、grid 并行、scatter/gather 支持、多候选 rollout。

## 参考来源

- Li et al., `BEVFormer`, ECCV 2022 / arXiv:2203.17270，https://arxiv.org/abs/2203.17270，成熟度：BEV 表征基础，查证日期：2026-05-29。
- Zheng et al., `BEVWorld: A Multimodal World Model for Autonomous Driving via Unified BEV Latent Space`, arXiv:2407.05679，https://arxiv.org/abs/2407.05679，成熟度：研究原型，查证日期：2026-05-29。
- Min et al., `DriveWorld: 4D Pre-trained Scene Understanding via World Models for Autonomous Driving`, CVPR 2024 / arXiv:2405.04390，https://arxiv.org/abs/2405.04390，成熟度：研究成熟，查证日期：2026-05-29。
- `DriveWAM: Video Generative Priors Enable Scalable World-Action Modeling for Autonomous Driving`, arXiv:2605.28544，https://arxiv.org/abs/2605.28544，成熟度：2026 前沿研究，查证日期：2026-05-29。

## 旧版素材

- `/mnt/e/workload-wiki-old/05_World_Model与生成式智能/BEV_World_Model.md`
