# LiDAR Point Cloud Processing

上级：[Vision and 3D Perception](README.md)
相关：[Multi-sensor Fusion](multi-sensor-fusion.md), [Occupancy Prediction](occupancy-prediction.md), [Occupancy Workload](../06-chip-workload-analysis/occupancy-workload.md)

## 这页在回答什么问题

LiDAR 点云处理回答如何从稀疏、不规则的 3D 点集合中提取几何和对象信息。它的核心 workload 挑战不是"3D 本身"，而是**不规则稀疏数据如何映射到偏好规整的 NPU**——这是整个稀疏计算与硬件矛盾的典型样本，理解它直接通向 06 的 [Occupancy Workload](../06-chip-workload-analysis/occupancy-workload.md)。

## 为什么它有效：直觉与类比

点云的直觉是**一把撒在 3D 空间里的反光图钉**：激光打到物体表面反射回来，每个回波是一个带坐标和强度的点 `(x,y,z,intensity)`，没打到的地方什么都没有。关键事实是——空间绝大部分是空的，图钉只钉在物体表面，所以点云天生极度稀疏。这个稀疏性既是机会也是麻烦，三种处理思路就是对"怎么对付稀疏"的三种回答。

**dense 3D conv** 是**给整个 3D 空间每个格子都派人检查**——把空间切成 voxel 网格，对每个 voxel 跑 3D 卷积。直觉上就知道亏：99% 的格子是空的，派去的人大半在空转。**sparse conv** 是**只在钉了图钉的格子干活**，但这里有个不直观的代价：卷积要读邻居，而稀疏存储下"谁挨着谁"不再是隐含的（dense grid 里邻居就是相邻下标），必须先建一本花名册（rulebook / indice pairs）记录"哪个 active voxel 参与哪个输出 voxel 的计算"，卷积才知道去 gather 哪些邻居。于是 sparse conv 省了空 voxel 的乘加，却换来 rulebook 构建、indice gather、负载不均这些新开销。**pillar/BEV** 是**把垂直方向压扁成柱子**——既然规划主要在地面坐标系，干脆把每根垂直柱内的点聚合成一个特征，3D 问题塌成 2D BEV，下游就能用规整的 2D conv（这也是 [BEV Perception](bev-perception.md) 的入口之一）。

这串对照揭示了点云处理的根本张力：**越保留原始点，几何表达越直接但越不规则；越投到 voxel/pillar/BEV，硬件越规整但越损失细节**。选哪种本质是在"几何保真"和"硬件友好"之间定位。

## 计算结构与稀疏的真实账

```text
raw points -> voxelization / pillarization -> sparse or BEV features -> 3D detection / segmentation / occupancy
```

point-based（PointNet 系）直接在点上做邻域搜索和聚合，保几何但硬件不规则；voxel-based（SECOND 系）量化到 3D grid 用 sparse conv；pillar（PointPillars）压高度换端侧效率。

把稀疏的账算清楚：自动驾驶单帧 LiDAR 有数万到十几万点。以 `X×Y×Z = 400×400×16` 的 dense grid 为例，理论 cell 超过 250 万，但真实 active voxel 可能只占百分之几——稀疏看起来能省几十倍计算。但省下的会被三样东西部分吃回去：rulebook 构建（每层都要重建邻接关系）、indice gather/scatter（按花名册去取邻居，是不规则访存，破坏 DRAM row locality），以及负载不均（点云在空间分布极不均匀，近处密、远处疏，导致并行单元忙闲不均）。

常见误解：稀疏就一定省。实际上稀疏省的是无效 voxel 的乘加，引入的是索引、metadata、负载不均、gather/scatter；如果硬件不支持稀疏调度，稀疏 workload 可能比 dense 还慢——这是架构师评估稀疏算子时必须算的总账。

## 一句话理解

LiDAR workload 的核心是"稀疏 3D 点 → 规整表示"的转换：它几何表达强，但 voxelization、sparse conv 的 rulebook、indice gather/scatter、负载不均共同把它推成 irregular-access-bound，省下的计算能否兑现完全取决于硬件的稀疏调度能力。

## 演进与未来方向（判断）

以下为判断，锚定真实趋势。

第一，**表示选择正在收敛到"统一的 BEV/voxel 共享空间"，点云 encoder 从独立模型变成喂给共享 3D 场景表示的一支**。早期 point/voxel/pillar 各自为战，现在多模态系统（BEVFusion 类，见 [Multi-sensor Fusion](multi-sensor-fusion.md)）把 LiDAR 特征投进和相机统一的 BEV，再往后和 Occupancy 的 voxel 空间合流（见 [Occupancy Prediction](occupancy-prediction.md)）。我的判断是独立的"LiDAR 检测模型"会越来越少，LiDAR 退化为统一 3D 场景表示的一个几何输入源——这让稀疏处理的位置从"端到端一条链"变成"前端 encoder 一段"，但稀疏访存的难题原封不动地搬进了共享空间。

第二条对架构师最关键：**稀疏 3D 是否值得专用硬件支持，是个该按目标产品显式权衡的取舍，而非普适结论**。纯视觉 BEV 方案（特斯拉路线）根本不用 LiDAR，sparse conv 一行用不上；重 LiDAR 的 Robotaxi 方案则把 sparse conv/voxelization 当主力算子。我的判断是：随着纯视觉占据法和 occupancy 兴起，部分量产端侧会绕开 sparse conv（用 pillar→2D 规避），而高等级自动驾驶/机器人仍强依赖原生稀疏支持。对 archax，sparse conv 应建模为 Interaction 轴上的 irregular-access 极端工作点，其**负载不均是建模的关键变量**（不能用平均 occupancy 估，要看空间分布的最坏情况），rulebook 构建是一段非计算非搬运的 metadata 开销——这正是 06 的 [Occupancy Workload](../06-chip-workload-analysis/occupancy-workload.md) 要量化的。

## Workload Characterization

计算密度：dense 3D conv 很重（大量空 voxel 空算）；sparse conv 减少无效计算但有效算术强度取决于稀疏调度质量；pillar/BEV 后接近规整 2D conv，算术强度回升。

访存模式：raw point 和 voxelization 是 scatter/gather；sparse conv 依赖 hash/rulebook 和邻接表的不规则索引访问，破坏 row locality；pillar/BEV feature 访问规则。

并行性：point、voxel、pillar 可并行；点云空间分布不均（近密远疏）导致严重负载不均，是稀疏并行的主要损耗。

数据复用：dense conv 有规则局部复用；sparse conv 复用受 occupancy pattern 影响、难预测；BEV projection 后复用恢复。

量化敏感度：BEV/CNN 部分适合 INT8；坐标量化、voxel indexing、几何聚合需保精度（坐标误差直接变几何误差）。

瓶颈类型：voxelization 和 sparse conv 常 irregular-access-bound；dense occupancy decoder 可能 capacity-bound；负载不均使最坏情况延迟远高于平均。

对硬件的核心需求：scatter/gather、sparse metadata/rulebook 支持、indexed load/store、负载均衡机制、BEV projection、2D/3D DMA——这套需求和规整 CNN 几乎正交，是 deterministic NPU 最难啃的一类，详见 [Occupancy Workload](../06-chip-workload-analysis/occupancy-workload.md)。

## 参考来源

- Qi et al., `PointNet: Deep Learning on Point Sets for 3D Classification and Segmentation`, CVPR 2017, arXiv:1612.00593。成熟度：已落地，point-based 出处。
- Yan et al., `SECOND: Sparsely Embedded Convolutional Detection`, Sensors 2018, arXiv:1806.05578。成熟度：已落地，sparse conv 检测代表。
- Lang et al., `PointPillars: Fast Encoders for Object Detection from Point Clouds`, CVPR 2019, arXiv:1812.05784。成熟度：已落地，pillar→2D 端侧高效路线。
