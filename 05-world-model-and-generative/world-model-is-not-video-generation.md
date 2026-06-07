# World Model Is Not Video Generation

上级：[World Model and Generative Intelligence](README.md)
相关：[World Model Fundamentals](world-model-fundamentals.md), [Video World Model](video-world-model.md), [JEPA and Self-supervised](../01-foundation-model-components/jepa-and-self-supervised.md), [World Model Workload](../06-chip-workload-analysis/world-model-workload.md)

## 这页在回答什么问题

这页是本章的思想核心，专门把一句话讲透：World Model ≠ 视频生成。视频可以是 World Model 的一种输出，但 World Model 的本质是 action-conditioned、状态一致、可度量安全的未来预测；混淆这两件事，会让架构师按"画质模型"的尺度去估算 workload，从而在 candidate 数、rollout horizon、可评估性这些真正决定成本的维度上系统性犯错。

## 为什么这个区分成立：直觉与类比

直觉是**电影特效师和飞行模拟器，做的是两件不同的事**。特效师追求"这帧看起来像真的"，哪怕物理上根本不成立——爆炸再绚丽，背后没有可查询的气压场。飞行模拟器追求"操纵杆这么打、飞机的状态会怎么变"，画面糙一点没关系，但每个状态量必须物理自洽、可被仪表读出、可用来判断会不会失速。视频生成是前者，World Model 是后者。对应到机制，前者优化的是像素分布的逼真度（人类偏好 / FID 这类感知指标），后者优化的是 action-conditioned 的状态演化是否一致、是否可被 collision/cost head 评估。

为什么"画得像"换不来"能决策"：一个视频模型可以把车流渲染得毫无破绽，但如果它的输出只是像素，planner 想知道"我这个 pose 会不会撞"，还得再上一个感知模型把像素解释回 object/occupancy/risk——而生成的几何只要漂移几十厘米，人眼能接受，collision metric 不能接受。对应到机制，这是因为像素空间里"逼真"和"几何正确"是两个不同的目标，优化前者不保证后者。这正呼应 [JEPA](../01-foundation-model-components/jepa-and-self-supervised.md)：与其逼模型把猜不准也用不上的纹理细节画对，不如让它在表示空间预测"会发生什么"的结构。

## 常见误解 → 实际上

误解一：World Model 就是会动作条件的视频生成。实际上 action-conditioning 只是必要条件之一；还要状态一致（多步 rollout 不漂移）、可控（能按结构化输入分支）、可评估（输出能接 collision/cost/risk head）。Sora、Veo 这类是强大的视频生成器，但默认不满足"可评估安全"这一条。

误解二：模型越大、画质越高，就是越好的 World Model。实际上一个单步 FLOPs 更大的视频模型，可能在"能否并发评估 64 个 candidate action、能否实时 rollout、能否输出结构化 state"上全面输给一个轻量 latent dynamics。评判 World Model 的尺度是 planning value，不是 FID。

误解三：视频是信息最完整的表示，所以最适合做 World Model。实际上信息完整恰恰意味着 workload 最重——T×H×W latent + 像素 decoder + 多步采样，把成本顶到云端；而决策需要的往往只是低维可决策结构，多出来的像素信息是为决策付了不必要的代价（这正是 latent/occupancy 表示的存在理由）。

## 为什么这个区分直接改变 workload

视频生成的 workload 由 high-resolution latent diffusion、spatial-temporal attention、像素 decoder 主导，是"宽而重复的生成计算"。World Model 在此之上（或之外）还叠加四样东西：action-conditioned rollout（沿时间迭代）、多 candidate 并发评估、latent/state cache 跨步驻留、collision/cost/risk head 与闭环调用。这四样决定了它的真实成本结构是 `candidate × horizon × per-step` 的乘法（见 [World Model Fundamentals](world-model-fundamentals.md)），而不是单帧画质模型的一次 forward。

对架构探索的直接后果：评估一个 World Model 能否端侧部署，不能只看单步 denoiser 的 FLOPs，要问三件事——是否需要实时 rollout（latency-bound）、是否要并发多个 candidate（cache 与并发度）、是否要输出结构化 state（决定能否省掉像素 decoder）。如果答案让你能用 latent 表示，端侧成本可能比对应的视频模型低一两个数量级；如果你被像素 decoder + 多步采样锁住，那它本质是云端 workload。

## 一句话理解

视频生成强调"未来看起来像真的"，World Model 强调"未来状态能用于决策"；两者重叠但不能混同——按画质尺度估 World Model 的 workload，会在 candidate、horizon、可评估性这些真正决定端侧可行性的维度上系统性失算。

## 演进与未来方向（判断）

以下为判断，锚定 2025-2026 联网核实的真实工作。查证日期：2026-06-07。

第一，**"逼真度"和"可决策性"两条评价线正在分离，且后者才是 World Model 的主战场**。2024-2026 视频生成（Sora、Veo、Genie 3）在长时一致与物理观感上进步飞快——Genie 3 已能 24fps、720p 实时交互、一致性维持数分钟。但 embodied AI 要的可控 World Model 仍卡在动作条件精度、交互一致性、可验证 safety metric 上。我的判断是：画质会继续逼近真实，但这不会自动让模型变成好的 World Model；真正的进展会来自给生成模型加结构化条件和可评估输出（GAIA-2 用 ego-action / 天气 / 道路语义等结构化条件 + 多视角一致就是这个方向）。

第二，这条判断落到硬件上：**端侧 World Model 的正确形态是"省掉像素、保留可评估状态"**，这把它从视频生成的重 workload 拉回 latent/occupancy 的轻 workload。V-JEPA 2 这类在 latent 空间预测、无像素 decoder 的路线，明确比像素生成式快一个量级，正是这个判断的证据。对 archax，这意味着不能把 World Model 一律按"视频生成器"建模——要按预测表示分流：像素生成式当云端 compute/capacity 大负载（连 [Cloud Inference and Simulation Chip](../06-chip-workload-analysis/cloud-inference-and-simulation-chip.md)），latent/occupancy 预测式当端侧带状态迭代 rollout（连 [World Model Workload](../06-chip-workload-analysis/world-model-workload.md)），两者在 Capability 和 Interaction 轴上是不同工作点，不能合并扫描。

## Workload Characterization

计算密度：视频生成偏重 diffusion/DiT denoise + 像素 decoder（compute + capacity 重）；World Model 在此之上叠加 action rollout 与 evaluator，但若用 latent/occupancy 表示则省掉像素 decoder，单步密度回落一两个数量级。

访存模式：视频模型有大 spatial-temporal latent（T×H×W）；World Model 还需 action/history/state cache 与 candidate buffer；latent 表示的 cache 远小于 video latent。

并行性：视频 sample 可并行；World Model 的 candidate action 可并行，但 closed-loop 的时间步严格依赖，是并行断点——这点视频续写也有，但 World Model 的多 candidate 评估让并行需求结构不同。

数据复用：World Model 复用同一历史 encoding 评估多个 action（`encode once → rollout many`）；纯视频生成的复用主要在 prompt/context，不存在多 candidate 共享 encoding 的省法。

量化敏感度：视频画质可容忍局部误差（人眼宽容）；World Model 的 collision/contact/occupancy 输出与长期一致性对误差敏感，安全相关维度需校验。

瓶颈类型：视频生成是生成吞吐 + 显存（capacity/compute-bound）；World Model 端侧还受实时性、candidate 数与 horizon 限制（latency-bound）——两者瓶颈类别不同。

对硬件的核心需求：可评估输出的结构化 head、action-conditioned 迭代 rollout、多 candidate 并发、condition 复用、（端侧）省掉像素 decoder 的 latent 路径——评判尺度是 planning value 而非画质，详见 [World Model Workload](../06-chip-workload-analysis/world-model-workload.md)。

## 参考来源

- OpenAI, `Video generation models as world simulators (Sora)`, 2024。成熟度：前沿视频生成系统展示，默认不满足可评估安全，查证日期：2026-06-07。
- Russell et al. (Wayve), `GAIA-2: A Controllable Multi-View Generative World Model for Autonomous Driving`, 2025, arXiv:2503.20523。成熟度：产业研究，结构化条件 + 多视角一致，区分"可控可评估"方向，查证日期：2026-06-07。
- Assran et al., `V-JEPA 2: Self-Supervised Video Models Enable Understanding, Prediction and Planning`, 2025, arXiv:2506.09985。成熟度：研究系统，latent 预测式 world model，无像素 decoder，查证日期：2026-06-07。
- Google DeepMind, `Genie 3: A New Frontier for World Models`, 2025-08。成熟度：前沿，实时交互生成、画质强但可评估性仍是开问题，查证日期：2026-06-07。
</content>
