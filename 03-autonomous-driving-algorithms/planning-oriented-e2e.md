# Planning-oriented E2E

上级：[Autonomous Driving Algorithms](README.md)
相关：[BEV Perception](../02-vision-and-3d-perception/bev-perception.md), [E2E Workload](../06-chip-workload-analysis/e2e-workload.md)

## 这页在回答什么问题

这页回答 planning-oriented E2E 为什么是自动驾驶算法路线的重要节点。它不是简单把感知-预测-规划串成一个大模型，而是用统一 BEV/query/token 表示，把多个中间任务和最终 planning loss 联合优化，让中间表征服务于最终驾驶轨迹而非各自的感知指标。

## 为什么它有效：直觉与类比

理解它要先看一个反直觉的事实：**每个感知子任务都做到 SOTA，最终规划未必最好**。把检测、建图、预测各自优化到 benchmark 第一，像一个团队里每个人都把自己的 KPI 刷满——检测组追 mAP、建图组追车道线精度——但没人对"车最终开得好不好"负责。结果可能是检测在远处小目标上抠出 0.5 个点（对规划无关紧要），却在"这个加塞车的运动意图"上不够准（对规划生死攸关），因为后者不在检测的 KPI 里。

planning-oriented E2E 的直觉就是**把所有中间任务的 KPI 重新绑定到同一个最终交付——"车该怎么开"**。UniAD 的做法是让检测、跟踪、建图、运动预测、occupancy 全都通过可微的 query 传递、最终汇到 planning，用规划质量当核心训练信号反向传导回去。于是每个中间任务学到的不再是"对它自己的 benchmark 最优"的表征，而是"对最终规划最有用"的表征——检测会自动更关注那些影响行车决策的目标，因为只有这些能降低 planning loss。对应到机制，关键是 query：object query、map query、motion query、planning query 在同一套 attention 里流动，让规划相关的信息能端到端地反传到每个上游任务（见 [BEV Perception](../02-vision-and-3d-perception/bev-perception.md) 的统一 query 框架）。

这带来两个本质变化：中间任务从"独立输出"变成"对 planning 有用的监督"；模型接口从离散 object list 扩展成 BEV feature、query state、vector map、future occupancy 等可微表示。VAD 进一步用 vectorized（向量化）场景表示替代过度 dense 的 BEV，用稀疏向量服务规划，换取效率——这是同一思想下的轻量化分支。

## 关键设计维度与工程边界

```text
multi-view cameras / lidar -> image/point encoder
   -> BEV / occupancy / vectorized scene representation
   -> detection + map + motion + occupancy + planning heads
   -> trajectory loss + auxiliary task losses
```

| 维度 | 选择 | 影响 |
| --- | --- | --- |
| 表示 | dense BEV、occupancy、vectorized、query set | 决定 memory footprint 和 head 结构 |
| 任务 | detection、map、tracking、prediction、planning | 决定监督复杂度与训练稳定性 |
| 时序 | short/long history、state cache | 决定 temporal cache 与 latency |
| 规划 | trajectory regression、candidate scoring、cost map | 决定输出可控性与安全约束接口 |

工程边界要讲清：planning-oriented E2E 不取消安全壳。量产系统仍保留 rule check、collision check、comfort constraint、fallback planner、监控模块。E2E 的价值是生成更一致的 scene/action 表征，不是替代所有系统安全机制——这点和 [Behavior Cloning E2E](behavior-cloning-e2e.md) 一致。

## 一句话理解

Planning-oriented E2E 把中间表征的优化目标从"服务感知 benchmark"重绑到"服务最终驾驶轨迹"，靠统一 query 让规划信号端到端反传；它把 workload 从单任务 backbone 推向共享表征、多 head 并发、时序状态和规划闭环。

## 演进与未来方向（判断）

以下为判断，锚定真实趋势。

主线判断：**planning-oriented E2E 正在和 VLM reasoning、World Model rollout 收敛成一个统一 policy**。UniAD/VAD 解决了"中间表征服务规划"，但它们的推理仍是几何/学习式的，缺少对长尾语义的理解和对"如果这样开会怎样"的显式推演。下一步很清楚——把 VLM 的开放语义推理（见 [VLM/VLA for AD](vlm-vla-for-ad.md)）和 World Model 的 action-conditioned rollout（见 [World Model for AD](world-model-for-ad.md)）并入这张可微图，让 policy 既懂场景语义又能内部模拟未来。这是 03 整章三条线（E2E、VLA、World Model）正在合流的终点。

对架构师，planning-oriented E2E 的 workload 画像有一个独特特征：**共享 BEV/query feature 的跨 head 复用是最大收益，但辅助任务越多、中间 activation 越大**。它不是一个 backbone 加一个 head，而是一个共享 encoder 喂多个并发 head（检测/建图/预测/occupancy/planning），调度上要支持多 head 并发、feature 跨 head 复用、且全程 batch=1。对 archax，这应建模为"共享 scene encoder（compute + irregular view transform）+ 多 head 并发（各自小但并行）+ temporal cache"的复合工作点，关键资源是 BEV feature 的片上驻留（决定能否跨 head 复用而不反复回 DRAM）和多 head 调度效率，而非单点峰值算力。随着 VLM/World Model 并入，还会叠加大模型 token 和 rollout 的新压力——这正是 06 [E2E Workload](../06-chip-workload-analysis/e2e-workload.md) 要系统刻画的多任务级联特征。

## Workload Characterization

计算密度：multi-view encoder、BEV transformer、query decoder、multi-task heads 共同构成主计算；planning head 通常小于 scene encoder。

访存模式：BEV feature、query state、map vector、occupancy grid、历史帧 cache 需跨 head 复用，读写路径复杂；view transform 段是 irregular access。

并行性：camera view、BEV cell、query、task head 可并行；tracking、motion、planning 之间存在依赖链，是并行断点。

数据复用：共享 BEV/query feature 是关键收益——辅助任务越多 feature reuse 越强，但中间 activation 也越大，对片上 buffer 容量提要求。

量化敏感度：backbone 和部分 MLP head 可低比特；geometry transform、attention score、trajectory regression、collision margin 对误差敏感。

瓶颈类型：训练侧是多任务 loss 与大 activation；推理侧是 BEV construction、temporal cache、多 head decoder latency，全程 batch=1。

对硬件的核心需求：高效 multi-view feature fusion、attention/query decode、BEV cache 管理（跨 head 复用的片上驻留）、低 batch 推理、多 head 并发调度——详见 [E2E Workload](../06-chip-workload-analysis/e2e-workload.md)。

## 参考来源

- Hu et al., `Planning-oriented Autonomous Driving (UniAD)`, CVPR 2023 / arXiv:2212.10156。成熟度：已落地研究，统一可微图代表（CVPR 2023 Best Paper）。
- Jiang et al., `VAD: Vectorized Scene Representation for Efficient Autonomous Driving`, ICCV 2023 / arXiv:2303.12077。成熟度：已落地研究，向量化高效分支。
- Caesar et al., `nuPlan: A closed-loop ML-based planning benchmark`, CVPR ADP3 2022 / arXiv:2106.11810。成熟度：常用闭环评测基准。
- Dauner et al., `NAVSIM: Data-Driven Non-Reactive Autonomous Vehicle Simulation and Benchmarking`, NeurIPS 2024 D&B / arXiv:2406.15349。成熟度：新兴评测基准。
