# Action Tokenizer

上级：[Robotics and VLA](README.md)
相关：[VLA Fundamentals](vla-fundamentals.md), [RT Series](rt-series.md), [VLA Workload](../06-chip-workload-analysis/vla-workload.md), [Attention and Transformer](../01-foundation-model-components/attention-and-transformer.md)

## 这页在回答什么问题

这页回答动作离散化/编码这件小事为什么是 VLA workload 的命门。语言和图像天然能 token 化，机器人动作却是连续、高频、带物理约束的信号；动作怎么编码，直接决定了"控制频率"这个最硬的实时指标是被 token 数卡死，还是被一次推理覆盖。这页只讲动作表示对计算图的影响，模型整体画像见 [VLA Fundamentals](vla-fundamentals.md)。

## 为什么它有效：直觉与类比

先看为什么要 tokenize。把动作离散成 token 的直觉是**把连续的舞步翻译成一套舞谱符号**，这样一个本来只会读写文字的人（语言模型）就能像续写句子一样把舞步"写"出来——动作和文字共用同一套生成机制，VLM 不用改结构就能输出控制量。对应到机制：把每个动作维度量化成有限个 bin，再把 bin 当作词表里的特殊 token，动作生成就变成了 next-token prediction，可以直接复用语言模型的 autoregressive decode 和交叉熵训练。这是 RT-2、OpenVLA 能"零改架构接动作"的全部秘密。

但朴素 binning 有个直觉上的坑：**舞谱记得太啰嗦**。OpenVLA 把 7 个自由度各量化成 256 个 bin，每个动作就是 7 个 token；如果还要一次出未来很多步（action chunk），token 数线性膨胀。对应到机制，autoregressive decode 是严格串行的——第 7 个 token 要等前 6 个出完，每一步都得把整个 backbone 权重过一遍 HBM——所以 token 数直接乘进控制周期的延迟里。一个 7B 模型 decode 一个 token 要几毫秒，7 个 token 就是几十毫秒，控制频率被压到个位数 Hz。这正是 RT-2-PaLI-X-55B 只能跑 1-3 Hz 的根。

FAST 的直觉是**别一个音符一个音符记，记成乐谱里的'和弦与节奏型'**——把一段动作序列变换到频率域，因为机器人动作在时间上高度相关、低频成分携带绝大部分信息，高频多是噪声。对应到机制：FAST 对每个动作通道做 DCT-II（type-II 离散余弦变换）把时序动作转成频域系数，量化后用 BPE（byte-pair encoding）把展平的系数序列压成更短的 token 串。相关性强的动作段在频域是稀疏的，于是同样一段动作，FAST 用的 token 数比逐维 binning 少得多——这就把"token 数 × decode 延迟"这条死结松开了。π0-FAST 正是靠它把 autoregressive VLA 扩到 10k 小时机器人数据、训练快约 5 倍，且性能追平 diffusion VLA。

flow/diffusion action head 则走完全相反的路：**不翻译成符号，直接生成连续舞步**。一个 action expert 以 backbone 隐藏状态为条件，从高斯噪声出发迭代去噪，一次输出未来一整段连续动作。对应到机制，它彻底绕过离散量化（没有 bin 边界的抖动），且一次推理出 H 步，但代价是迭代去噪本身要跑多步。这条路不产生"token"，所以它的 workload 性格和前两者完全不同——下文会把这点钉清楚。

## 三种表示的计算图，各卡在哪

把三条路线的瓶颈摊开，才能判断一个 VLA 的 workload 性格。

离散 binning（RT-2 / OpenVLA）：动作 = 维度数 × chunk 步数 个 token，串行 autoregressive decode。瓶颈是 **token 数 × 单 token decode 延迟**，且单 token decode 在 batch=1 下是 memory-bandwidth-bound（每 token 过一遍全部权重）。OpenVLA 单臂 7-DoF 单步动作是 7 token，控制频率受限于 backbone 大小；要提频就得砍 token 数或砍模型。

FAST 频域 tokenization：先 DCT 再 BPE，把动作 chunk 压成显著更短的 token 串。瓶颈仍是 autoregressive decode，但 token 数降一个量级后，同样的 backbone 能跑更高频或更长 chunk。额外成本是编解码端的 DCT/BPE（轻量，CPU 量级），以及训练时需要先拟合 BPE 词表（FAST+ 是在 100 万条真实动作轨迹上训出的通用 tokenizer，可黑盒套用到不同机器人）。

flow/diffusion action expert（π0 / GR00T N1）：无 token，action expert 从噪声迭代去噪出 H 步连续动作。瓶颈是**去噪步数 × action expert 单步计算**，而不是 token 串行——这是个可并行（chunk 内 H 步一次生成）但有迭代深度的计算。π0 的 300M expert 在 RTX GPU 上约 73ms 出 50 步动作，GR00T N1 的 DiT 出 16 步动作，整模约 64ms 出一个 chunk。它把瓶颈从"token 数"换成了"去噪迭代数"，对量化和数值稳定性的敏感点也随之转移。

一个贯穿的取舍：action chunk 越长，单位时间推理次数越少（省算力、提有效控制频率），但闭环反馈变慢——chunk 里第 H 步动作是基于 H 步前的 observation 算的，环境若在 chunk 中途变了，模型要到下个 chunk 才知道。所以 chunk 长度是"实时反应性 vs 推理频率"的旋钮，不是越长越好。

## 一句话理解

Action Tokenizer 决定 VLA 是"长 token 序列的串行生成问题"（binning，受 token 数 × decode 延迟卡死）、"短频域 token 问题"（FAST，DCT+BPE 把 token 数压一个量级），还是"迭代去噪的连续生成问题"（flow/diffusion，瓶颈换成去噪步数）；它是控制质量和硬件成本之间的接口，也是控制频率这个硬指标的直接决定者。

## 演进与未来方向（判断）

以下为判断，锚定 2025-2026 联网核实的真实工作，查证日期：2026-06-07。

演进脉络是**从"逐维 binning"走向"频域压缩 token"和"连续 action expert"两条并行的高效路线**。2023 的 RT-2 还在用最朴素的整数 token（动作 bin 映射到词表里 0-1000 的整数 token）；2025 的 FAST（arXiv:2501.09747）用 DCT+BPE 把动作 token 数压一个量级，并放出在 100 万条轨迹上训练的通用 tokenizer FAST+，让"高效动作 token"从每个项目自己调，变成可黑盒复用的基础设施。与此同时 π0/π0.5、GR00T N1 这条 flow-matching 线干脆不用 token。

我的几条判断。其一，**离散 token 路线不会消失，但会几乎全部迁到频域压缩（FAST 类）**，因为它保留了"复用语言模型 autoregressive 机制"这个最大工程便利，只要把 token 数压下来就能和 flow 路线在控制频率上掰手腕——纯逐维 binning 在 2026 已基本只剩教学价值。其二，**flow/diffusion action expert 会是高频精细操作的主力**，因为它在控制频率（一次出整段 chunk）和动作多模态表达（避开量化、能表示"有几种合理动作"）上都占优，Helix 的 200 Hz、GR00T 的 120 Hz System 1 都是这条路。其三，**tokenizer 正在从"模型的一部分"变成"跨 embodiment 的标准接口"**——FAST+ 这种黑盒通用 tokenizer 意味着同一套动作编码能服务不同自由度、不同控制频率的机器人，这对跨本体数据混训是关键基础设施。

对架构师，这条演进直接改写 action head 的 workload 性格，且这正是 06 [VLA Workload](../06-chip-workload-analysis/vla-workload.md) 要刻画的。两条路线对硬件提的需求是不同的：FAST 路线仍是 backbone autoregressive decode 主导（memory-bandwidth + KV cache），只是 token 数少了；flow 路线把一部分负载从 backbone decode 转移到 action expert 的迭代去噪——这是一个固定迭代次数、chunk 内可并行的小模型反复跑，对 archax 而言是一个和 LLM decode 性格不同的计算 phase，应作为独立工作点建模。换言之，"这个 VLA 用什么 action tokenizer"决定了它在 archax 里属于哪一类 workload 剖面，不能笼统按"大模型推理"对待。

## Workload Characterization

以 7-DoF 单臂、224px 单相机、batch=1 端侧为基准，分三种表示刻画。

计算密度：tokenizer/解码器本身都很轻（DCT/BPE 是 CPU 量级，VQ/binning 是查表）；真正的计算来自动作表示导致的 backbone 负载差异——binning 是 token 数 × 单 token decode（7B backbone 单 token 约 14 GFLOP），FAST 把 token 数压一个量级、总 decode 量随之降，flow expert 是去噪步数（典型几步到几十步）× 300M 级 expert 单步。

访存模式：autoregressive 路线（binning/FAST）的动作 token decode 严重依赖 KV cache 读写，每 token 过一遍 backbone 权重；flow expert 路线则是 action expert 权重 + 短窗口 action/state 缓存的反复读取，KV 压力小但多了去噪迭代的中间张量。

并行性：连续 regression 一次并行输出；离散 token 严格串行（FAST 减的是串行长度不是串行性本身）；flow/diffusion 在 chunk 内 H 步可一次并行生成，但去噪 step 之间有迭代依赖——这是"批并行强、迭代深度不可省"的结构。

数据复用：同一份 observation encoding 可复用于多个 action candidate（采样多条动作再筛）；action chunk 让一次视觉编码覆盖多个控制周期（π0 的 H=50 在 50Hz 机器人上覆盖约 1 秒），是降低视觉编码频率的关键复用。

量化敏感度：token embedding 和 backbone 可量化；但接触阶段、gripper 开闭、末端姿态的动作边界对精度敏感——binning 路线敏感在 bin 边界量化，flow 路线敏感在去噪数值稳定性，两者敏感点不同但都需任务级验证。

瓶颈类型：高频控制下，**token decode latency（binning/FAST）或去噪迭代延迟（flow）比单次 FLOPs 更决定能否闭环**；chunk 长度是反应性与推理频率的旋钮；训练侧还受动作数据归一化（如 OpenVLA 的 1-99 百分位裁剪）和跨机器人动作空间对齐影响。

对硬件的核心需求：低延迟小 batch decode（autoregressive 路线）或低延迟迭代去噪（flow 路线）、KV cache 高效访问、短序列实时调度、连续动作后处理（反量化/逆 DCT）、与低层 kHz 控制器的稳定交接——详见 [VLA Workload](../06-chip-workload-analysis/vla-workload.md)。

## 参考来源

- Chi et al., `Diffusion Policy: Visuomotor Policy Learning via Action Diffusion`, RSS 2023 / arXiv:2303.04137。成熟度：已落地研究路线，连续动作扩散生成代表。查证日期：2026-06-07。
- Zitkovich et al., `RT-2`, CoRL 2023 / arXiv:2307.15818。成熟度：已落地研究，动作即整数 text token（0-1000）的早期代表。查证日期：2026-06-07。
- Kim et al., `OpenVLA`, arXiv:2406.09246。成熟度：开源 baseline，7-DoF 各 256 bin、覆写 Llama-2 最少用的 256 个 token。查证日期：2026-06-07。
- Pertsch et al. (Physical Intelligence), `FAST: Efficient Action Tokenization for Vision-Language-Action Models`, arXiv:2501.09747。成熟度：2025 研究阶段，DCT-II + BPE 频域 tokenization，FAST+ 在 100 万条轨迹上训练的通用 tokenizer。查证日期：2026-06-07。
- Black et al. (Physical Intelligence), `π0`, arXiv:2410.24164。成熟度：研究阶段，flow-matching action expert（一次出 H=50 步）。查证日期：2026-06-07。
- `SmolVLA: A Vision-Language-Action Model for Affordable and Efficient Robotics`, arXiv:2506.01844。成熟度：2025 研究阶段，高效可部署 VLA。查证日期：2026-06-07。
