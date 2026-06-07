# Behavior Cloning E2E

上级：[Autonomous Driving Algorithms](README.md)
相关：[E2E Workload](../06-chip-workload-analysis/e2e-workload.md), [Video Understanding](../02-vision-and-3d-perception/video-understanding.md), [Diffusion Models](../01-foundation-model-components/diffusion-models.md)

## 这页在回答什么问题

这页回答 Behavior Cloning（BC）式端到端自动驾驶是什么、为什么重新受关注、它的硬问题在哪。重点不是把 E2E 当黑盒，而是看它如何把感知-预测-规划-控制的接口压缩成"从 observation 到 action"的学习问题，以及为什么它的难点从感知精度转移到了分布偏移和闭环稳定。

## 为什么它有效，又为什么会崩：直觉与类比

BC 的直觉极简单——**看着老司机怎么开，照着模仿**。给模型喂大量"这个场景下人类打了这个方向、踩了这个刹车"的数据，让它学这个映射。像学徒看师傅干活，看得够多就照葫芦画瓢。这为什么诱人：训练接口简单、数据能随车队规模无限扩展，而且能把那些根本写不出规则的东西——交互让行的微妙时机、加塞时的路权博弈、老司机的"手感"——直接当成学习目标，绕开了手写规则的天花板。

但同一个"照着模仿"的设计里藏着 BC 最致命的病，叫 **covariate shift（协变量偏移）**，直觉是这样：**学徒只在师傅走过的"正确路线"附近见过世面，从没见过偏出这条线之后该怎么办**。师傅开得太好，永远不会让车斜着冲向路肩，于是训练数据里压根没有"车正斜冲向路肩"这种状态。可学徒自己上路时，难免有个微小误差让车偏出正轨一点点——这个状态它没见过，于是输出一个更离谱的动作，车偏得更多，进入更没见过的状态……**误差像滚雪球一样自我放大**。这是 BC 的结构性缺陷：它学的是"专家轨迹上的动作"，而不是"从任何状态恢复到安全的能力"，一旦离开专家分布就失去依靠。这解释了为什么纯 open-loop 看着完美的 BC 模型，一上闭环就可能失稳。

第二个机制性问题是**驾驶决策天然多峰，而朴素回归会把多峰平均成灾难**。前方有障碍，左绕和右绕都对，但如果模型用回归去拟合"平均动作"，平均的结果是直接撞上去。这正是 [Diffusion Models](../01-foundation-model-components/diffusion-models.md) 和 action sampling 在驾驶里有价值的原因——多步去噪能走向某一个合理模式而非把它们糊成中间值，所以现代 BC 越来越多用 diffusion policy 或离散 action token 来保住动作分布的多峰性，而不是单点回归。

## 基本形式与输出接口

```text
multi-camera / lidar / route / ego state -> scene encoder + temporal model
   -> trajectory / waypoint / control token -> imitation loss
```

早期路线是 camera 直接到 steering/throttle/brake（PilotNet、ChauffeurNet）；新一代通常不再直接输出低维控制，而是输出 future waypoints、ego trajectory、action token，交给低层控制器消费。输出接口大致四类，workload 影响不同：

| 输出形式 | 特点 | workload 影响 |
| --- | --- | --- |
| 控制量 | latency 低，可解释性弱 | decoder 极轻，压力全在 encoder |
| waypoint / trajectory | 当前最常见，便于控制与评估 | trajectory head 小，需时序稳定 |
| cost / occupancy guided action | 与传统 planner 结合 | 需额外 cost map / future state |
| action token | 适合 VLA / sequence model | decoder 变 autoregressive，带串行 decode |

## 与 planning-oriented E2E 的差别

planning-oriented E2E（见 [Planning-oriented E2E](planning-oriented-e2e.md)）保留检测、map、motion、occupancy 等中间监督，让规划 head 成为统一任务之一；BC 更强调最终行为模仿，可弱化中间任务、甚至只用 action loss 加少量 safety auxiliary loss。这不是二选一——工程系统常用中间任务稳定表征，再用 imitation/action loss 把 representation 拉向驾驶决策。

## 一句话理解

Behavior Cloning E2E 把自动驾驶从"手写模块接口"推向"从驾驶数据学习行为接口"，靠模仿绕开规则天花板；但它的硬问题从感知精度转移到了 covariate shift、闭环稳定、多峰动作和长尾恢复——这些都不是 open-loop 指标能暴露的。

## 演进与未来方向（判断）

以下为判断，锚定真实趋势。

主线判断：**纯 open-loop BC 已到平台期，未来是"BC 预训练 + 闭环/RL/World Model 精炼"的组合**。covariate shift 决定了光靠模仿专家轨迹无法学到恢复能力，必须用闭环纠偏——具体手段正在收敛到几条：closed-loop 训练与评估、trajectory perturbation 制造离分布状态再教模型恢复、RL fine-tuning（2025 的 AutoVLA 就用强化微调，见 [VLM/VLA for AD](vlm-vla-for-ad.md)）、以及用 World Model rollout 在生成的未来里训练 policy（见 [World Model for AD](world-model-for-ad.md)，这是直接对治 covariate shift 的路子——在仿真的偏离状态里学恢复）。我的判断是 BC 会长期作为"用海量车队数据打底"的预训练阶段存在，但单独拿它做最终 policy 会被闭环方法补强。

对架构师，BC 的车端 workload 画像很具体且和后面 VLA 一脉：multi-camera encoder + temporal cache 主导（见 [Video Understanding](../02-vision-and-3d-perception/video-understanding.md)），trajectory head 本身很轻，**真正的硬约束是 batch=1 实时推理和异常帧下的 deterministic fallback**——闭环系统里一帧延迟抖动或一次非确定输出都可能触发雪球。对 archax，BC 应建模为"重 encoder + 轻 decoder + 历史状态 cache"的工作点，其关键不是峰值算力而是 batch=1 的稳定低延迟和确定性；若输出 action token，则叠加 autoregressive decode 的串行尾巴。这是 06 [E2E Workload](../06-chip-workload-analysis/e2e-workload.md) 的核心场景之一。

## Workload Characterization

计算密度：多摄像头 encoder 和 temporal attention/recurrent state 是主计算；trajectory head 通常很轻。

访存模式：video frame buffer、multi-view feature、ego state、route token、历史 latent 需持续缓存；长历史放大带宽与容量。

并行性：camera view、frame、feature stage 可并行；autoregressive action token 或 closed-loop rollout 引入串行依赖。

数据复用：同一 scene feature 可复用给 imitation loss、auxiliary perception loss、safety loss、uncertainty head。

量化敏感度：image encoder 较适合量化；action decoder、trajectory regression、small-object/traffic-light 相关特征需更谨慎（直接关乎安全）。

瓶颈类型：训练侧多为数据吞吐和 video IO；车端推理常由 multi-camera encoder + temporal cache 决定 latency，且对延迟抖动和非确定性敏感（闭环雪球）。

对硬件的核心需求：稳定多路视频输入、历史状态缓存、低延迟 trajectory decode、batch=1 推理效率、异常帧下的 deterministic fallback——确定性比峰值算力更关键，详见 [E2E Workload](../06-chip-workload-analysis/e2e-workload.md)。

## 参考来源

- Bojarski et al., `End to End Learning for Self-Driving Cars (PilotNet)`, arXiv:1604.07316。成熟度：早期范式。
- Bansal et al., `ChauffeurNet: Learning to Drive by Imitating the Best and Synthesizing the Worst`, RSS 2019 / arXiv:1812.03079。成熟度：经典闭环 imitation baseline，已用合成扰动对治 covariate shift。
- Wu et al., `Trajectory-guided Control Prediction for End-to-end Autonomous Driving (TCP)`, NeurIPS 2022 / arXiv:2206.08129。成熟度：研究成熟，强 baseline。
- Chi et al., `Diffusion Policy: Visuomotor Policy Learning via Action Diffusion`, RSS 2023 / arXiv:2303.04137。成熟度：已落地研究，多峰动作分布代表。
