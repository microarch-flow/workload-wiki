# Multi-sensor Fusion

上级：[Vision and 3D Perception](README.md)
相关：[BEV Perception](bev-perception.md), [LiDAR Point Cloud Processing](lidar-point-cloud-processing.md), [BEV Workload](../06-chip-workload-analysis/bev-workload.md)

## 这页在回答什么问题

多传感器融合回答如何把 camera、LiDAR、radar、IMU 等不同模态对齐到同一个 scene representation。它的 workload 本质是多路并行 encoder、坐标对齐、特征投影、跨模态融合——而真正的难点和瓶颈不在"融合算子"本身，在对齐和同步。这页要讲清为什么 fusion stage 常是整条 pipeline 的 p99 latency 决定点。

## 为什么它有效：直觉与类比

融合的直觉是**让几个各有所长但各说各话的证人，在同一张桌子上对齐口供**。相机是懂语义的证人（看得出"那是红灯""那是行人"，但说不准距离）；LiDAR 是懂几何的证人（能精确报出"前方 23.4 米有个东西"，但说不清那是什么）；radar 是懂速度和恶劣天气的证人（雨雾里仍能测出"有个目标以 30km/h 接近"，但分辨率粗）。单靠任一个都有致命盲区，合起来才完整——这是融合有效的根本：**模态互补，各补对方的盲区**。

但这里有个被名字误导的关键点。"融合"听起来像是把证词拼在一起（concat），其实**拼接是最不重要的一步，难的是让证词可对齐**。三个证人用不同坐标系（相机是像素平面、LiDAR 是 3D 车体系）、不同采样频率（相机 30Hz、LiDAR 10Hz）、不同时间戳、不同分辨率、不同置信度在说话，还可能有证人临时缺席（相机被泥糊住、LiDAR 雨天失效）。融合系统真正的工作量在把这些异步、异构、异坐标的证词校准到可比较——对应到机制，这就是为什么 fusion 的成本压在 projection、calibration、temporal alignment，而不是那个 concat/attention 算子。

为什么 BEV 成了对齐用的"那张桌子"：因为规划本来就在地面俯视坐标系里做决策。把所有模态都投到 BEV，等于让融合坐标系和下游规划坐标系合一，一举解决了透视畸变、多相机重叠、模态坐标不一致——这是 [BEV Perception](bev-perception.md) 在 02→03 桥梁位置的根本理由。

## 计算结构与那个同步瓶颈

```text
camera / lidar / radar -> modality-specific encoders -> projection / calibration / alignment
   -> fusion in image / point / BEV / token / object space -> joint scene representation
```

按融合位置分四类，各有 workload 取舍。early fusion 在原始数据附近融合，信息交互最充分但标定对齐最复杂（频率、坐标、分辨率、噪声模型全不同）；late fusion 在结果级合并，工程简单但每个模态已丢掉中间不确定性，互补信息利用不足；mid-level（feature-space）fusion 是自动驾驶最常见路径，在语义和几何间折中，代价是中间 feature 搬运和 alignment 成本高；BEV-space fusion 是 2022 之后的主线，把各模态投到统一鸟瞰坐标系融合。

最该被架构师记住的是这条：**即使各模态 encoder 都能并行跑，fusion stage 仍必须等到时间对齐后的 feature 齐了才能动**。于是整条 pipeline 的 p99 latency 常由最慢的那个模态、最大的那次 feature 搬运、或一个 CPU/NPU 同步点决定，而不是由平均吞吐决定。这是个典型的"木桶最短板"结构——优化平均算力没用，得优化最慢支路和同步开销。

常见误解：融合就是 concat。实际上真正难的是坐标系、时间戳、分辨率、置信度、缺失模态的对齐；concat 只是融合算子的表面形式，对齐才是工作量所在。

## 一句话理解

多传感器融合靠模态互补补齐各自盲区，把多路异构输入变成统一 scene representation；它的核心 workload 不是融合算子而是多路并行 encoder + 投影对齐 + 时间同步，且 fusion stage 是 pipeline 的汇合点，p99 latency 由最慢模态和同步点决定。

## 演进与未来方向（判断）

以下为判断，锚定真实趋势。

第一，**融合正全面收敛到 BEV/统一 token 空间，并继续往 end-to-end sensor-to-plan 推进**。从 late/feature fusion 到 BEVFusion，再到把融合后的 BEV/token 直接喂给 E2E 规划（见 [03 章](../03-autonomous-driving-algorithms/README.md)），趋势是融合不再产出"检测结果"而是产出"共享场景表示"。我的判断是融合 stage 会越来越深地嵌入统一 scene transformer，concat/对齐这类操作被 cross-attention 式的 query 融合取代（BEV query 直接从各模态采样）。但有一条不会变：**无论架构怎么演进，融合始终是 pipeline 里必须等齐多路输入的同步点**，它是 p99 的结构性来源，不是某个算子能消除的。

第二条对架构师最实际：**缺失模态的鲁棒性正成为硬指标**。量产系统必须在某个传感器失效时优雅降级（相机糊了靠 LiDAR、LiDAR 雨天失效靠相机+radar），这意味着融合不能假设所有模态总在线。对 archax 的含义：融合在 Topology/Interaction 轴上是一个**多流汇聚的同步节点**，建模时必须显式表达"等齐多路 + 时间对齐 buffer + 缺失模态分支"，而不能当成一次普通算子；它的关键资源是多路 DMA 带宽和同步开销，关键风险是最慢支路——这正是 06 的 [BEV Workload](../06-chip-workload-analysis/bev-workload.md) 要量化的 pipeline 级特征。这也和 [Multi-sensor Fusion 的稀疏侧](lidar-point-cloud-processing.md) 的负载不均、[Neck](../01-foundation-model-components/neck-feature-fusion.md) 的多分支同步是同一类"非计算主导、同步主导"的 workload。

## Workload Characterization

计算密度：各模态 encoder 可能 compute-bound；alignment/projection/concat/attention fusion 常 memory 或 irregular-access-bound——融合本身很少是算力瓶颈。

访存模式：camera feature 规则、体量大，LiDAR/radar 稀疏；跨模态投影带来 gather/scatter 和 layout conversion；时间对齐需要历史 feature buffer。

并行性：不同传感器 encoder 可并行；fusion stage 必须同步，受最慢模态和时间对齐约束——这是并行性的硬断点。

数据复用：单模态 feature 可在多任务 head 复用；BEV feature 可作共享 scene cache 跨任务、跨帧复用。

量化敏感度：encoder 可低比特；几何投影、calibration、depth/coordinate 相关部分需保精度（标定误差直接放大成融合错位）。

瓶颈类型：典型瓶颈是 bandwidth、synchronization、irregular access、p99 latency；fusion stage 是多路 pipeline 的汇合点，决定尾延迟。

对硬件的核心需求：多路并行 DMA、时间同步 buffer、scatter/gather、BEV feature cache、跨模态 fusion kernel，以及对缺失模态的降级路径支持——详见 [BEV Workload](../06-chip-workload-analysis/bev-workload.md)。

## 参考来源

- Liang et al., `Deep Continuous Fusion for Multi-Sensor 3D Object Detection`, ECCV 2018。成熟度：已落地，feature-level 融合早期代表。
- Liu et al., `BEVFusion: Multi-Task Multi-Sensor Fusion with Unified Bird's-Eye View Representation`, ICRA 2023 / arXiv:2205.13542。成熟度：已落地，BEV 空间多任务融合代表。
- Liang et al., `BEVFusion: A Simple and Robust LiDAR-Camera Fusion Framework`, NeurIPS 2022 / arXiv:2205.13790。成熟度：已落地，强调缺失模态鲁棒性的 LiDAR-相机融合。
