# BEV World Model

上级：[World Model and Generative Intelligence](README.md)
相关：[BEV Perception](../02-vision-and-3d-perception/bev-perception.md), [World Model for Autonomous Driving](../03-autonomous-driving-algorithms/world-model-for-ad.md), [BEV Workload](../06-chip-workload-analysis/bev-workload.md), [World Model Workload](../06-chip-workload-analysis/world-model-workload.md)

## 这页在回答什么问题

这页回答 BEV World Model 为什么是自动驾驶 planning 友好的那一支：它在已经几何对齐的鸟瞰平面上预测未来，输出直接可被 planner 和 rule check 使用。重点是讲清"在 BEV 上 rollout"相比视频和纯 latent 各换来什么、又各放弃什么，以及它的 workload 为什么由 BEV grid 尺寸、时序对齐和 candidate 数共同决定，而不是 BEV 感知的网络细节（那在 02 章）。

## 为什么它有效：直觉与类比

BEV World Model 的直觉是**在一张俯视的战场沙盘上推演，而不是从每个士兵的第一视角看录像**。指挥官摆沙盘是因为俯视图天然把距离、朝向、相对位置摊平成可直接量度的二维布局——谁离谁多远、这条路线会不会撞，一眼可读。BEV 就是自动驾驶的这张沙盘：多相机 + LiDAR 已被对齐到 ego-centric ground plane，距离、速度、轨迹、碰撞关系都在同一坐标系里。对应到机制，World Model 在 BEV feature/grid 上做 temporal dynamics，预测 future BEV feature / actor motion / lane occupancy / cost map，输出本身就在 planner 工作的坐标系里。

为什么这比视频更适合规划：视频续写出的是像素，planner 还要再上一个感知模型把它解释回 object/risk；BEV 预测的是已对齐的空间状态，省掉这道解释，且不追求逼真像素，没有像素 decoder 和多步采样的重负担。对应到机制，预测对象从像素换成 BEV grid，single-step 从大生成模型塌缩成 BEV encoder + temporal dynamics，成本落在视频与纯 latent 之间。

为什么不全用纯 latent：BEV 比抽象 latent 可解释、可接 rule check——一个 future BEV cost map 能逐格检查，一团 latent 不能。对应到机制，BEV 保留了显式空间结构，这让安全约束和 planner 接口更直接。代价是 BEV 仍是 2D：高度、遮挡、悬空物体、坡道表达不全，复杂 3D 占用要交给 [Occupancy World Model](occupancy-world-model.md)。

## 基本结构与 workload 的三个决定因素

```text
history sensor frames
   -> BEV encoder (view transform: 多相机/LiDAR -> BEV feature)
   -> BEV temporal dynamics  [沿时间 + ego-motion 对齐, rollout H 步]
   -> future BEV feature / actor motion / lane occupancy / cost map
   -> planner / simulator / risk head
```

它的 workload 由三件事共同决定，缺一不可：BEV grid 尺寸（如 200×200 的 cell 数直接定 feature map 容量与每步计算）、时序与 ego-motion 对齐（每步要把历史 BEV 按自车运动 warp 到当前坐标，是持续的 scatter/gather 与 resample）、candidate × horizon 乘法（多候选轨迹各跑一条 BEV rollout）。前端 view transform + BEV cache 常是 bottleneck（与 02 [BEV Perception](../02-vision-and-3d-perception/bev-perception.md) 同源），future rollout 的瓶颈则看 horizon 和 candidate 数。

代表工作的两种做法值得区分。一种是判别式 BEV 预测（DriveWorld 这类 4D 预训练场景理解，直接预测 future BEV/occupancy 表征服务下游），输出轻、贴近感知 head；另一种是生成式 BEV（BEVWorld，用 multimodal tokenizer 把多模态压进统一 BEV latent，再用 latent BEV sequence diffusion 在 action token 条件下预测未来场景），表达力强但把 diffusion 的多步采样成本带进 BEV——同样是 BEV 表示，workload 可以差出一个采样步数的乘子。

## 一句话理解

BEV World Model 在几何已对齐的鸟瞰沙盘上预测未来，用显式空间结构换可评估性与 planner 友好；它比视频轻（省像素 decoder）、比纯 latent 可解释，workload 由 BEV grid 尺寸、ego-motion 时序对齐和 candidate × horizon 乘法共同决定，但受限于 2D 不能完整表达高度与遮挡。

## 演进与未来方向（判断）

以下为判断，锚定 2024-2026 真实工作。查证日期：2026-06-07。

第一，**BEV World Model 正从"判别式未来预测"向"统一多模态生成式 BEV"演进，但端侧要的是前者**。DriveWorld（CVPR 2024）代表判别式路线，输出贴近 planning 与感知；BEVWorld（2024，latent BEV diffusion）代表生成式路线，把图像与点云统一进共享 BEV latent、可双向 encode/decode 并条件于 action 预测未来。我的判断是：生成式统一 BEV 在云端做数据生成与预训练很有价值，但端侧实时规划会用判别式或极少步的 BEV 预测——因为 BEV 上一旦引入 diffusion 多步采样，candidate × horizon × steps 的乘法又会把成本顶起来。

第二，落到硬件上：**BEV World Model 的端侧画像 = BEV encoder（含 view transform 的 scatter/gather）+ 时序 ego-motion 对齐 + 轻 future head + 多 candidate rollout**，它和 02 的 BEV 感知共享前端 bottleneck，但多出"沿时间维护并 warp BEV state"这一带状态迭代。对 archax，BEV grid 分辨率、horizon、candidate 数是必须显式扫描的维度，view transform 的 scatter/gather 和 ego-motion warp 是相对纯 GEMM 的特殊算子需求；这衔接 06 [BEV Workload](../06-chip-workload-analysis/bev-workload.md)（前端）与 [World Model Workload](../06-chip-workload-analysis/world-model-workload.md)（rollout 乘法）。BEV World Model 的 rollout 与 [Mamba 状态递推](../01-foundation-model-components/mamba-and-ssm.md) 同属带状态的迭代推理这条主线。

## Workload Characterization

计算密度：BEV encoder（含 view transform）+ temporal dynamics 是主计算；future head 通常比视频 decoder 轻一个量级；若用生成式 BEV diffusion，单步再乘采样步数，密度向视频侧靠拢。

访存模式：BEV feature map（如 200×200×C）+ history BEV cache + ego-motion alignment + future horizon 产生持续读写；view transform 是 scatter/gather，ego-motion warp 是 resample，访存不规则。

并行性：BEV cell、future candidate、task head 可并行；temporal rollout 沿时间依赖（含逐步 ego-motion warp）是并行断点。

数据复用：同一 BEV feature 可同时服务 detection、motion、planning 与 world model rollout；history BEV 与 map prior 跨 future step 和 candidate 复用（`encode once → rollout many`）。

量化敏感度：BEV backbone 可量化；几何对齐、collision boundary、trajectory cost 对误差敏感，且误差沿 rollout 与逐步 warp 累积。

瓶颈类型：前端常是 view transform + BEV cache（bandwidth/scatter-gather-bound，与 02 BEV 感知同源）；future rollout 的瓶颈取决于 horizon、candidate 数，生成式 BEV 还叠加采样步数。

对硬件的核心需求：BEV feature 缓存、view transform 的 scatter/gather、ego-motion warp/resample、grid 并行、多 candidate rollout 调度——详见 [BEV Workload](../06-chip-workload-analysis/bev-workload.md) 与 [World Model Workload](../06-chip-workload-analysis/world-model-workload.md)。

## 参考来源

- Li et al., `BEVFormer: Learning Bird's-Eye-View Representation from Multi-Camera Images via Spatiotemporal Transformers`, ECCV 2022 / arXiv:2203.17270。成熟度：已落地，BEV 表征基础。
- Min et al., `DriveWorld: 4D Pre-trained Scene Understanding via World Models for Autonomous Driving`, CVPR 2024 / arXiv:2405.04390。成熟度：研究成熟，判别式 4D BEV/occupancy 预训练。
- Zhang et al., `BEVWorld: A Multimodal World Simulator for Autonomous Driving via Scene-Level BEV Latents`, 2024 / arXiv:2407.05679。成熟度：研究原型，多模态 tokenizer + latent BEV sequence diffusion，action 条件，查证日期：2026-06-07。
- Wang et al., `Driving into the Future: Multiview Visual Forecasting and Planning with World Model (Drive-WM)`, CVPR 2024 / arXiv:2311.17918。成熟度：研究成熟，可与 E2E planning 结合。
</content>
