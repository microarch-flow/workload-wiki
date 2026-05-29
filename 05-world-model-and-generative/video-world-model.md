# Video World Model

上级：[World Model and Generative Intelligence](README.md)
相关：[World Model Is Not Video Generation](world-model-is-not-video-generation.md), [World Model Workload](../06-chip-workload-analysis/world-model-workload.md)

## 这页在回答什么问题

这页回答 Video World Model 的价值和边界。它通过预测未来图像或视频，让系统获得可视化的未来模拟；但它只有在 action-conditioned、时序一致、可被评估时，才对自动驾驶和机器人真正有用。

## 计算结构

```text
history frames + optional action / text / map
   ->
spatial-temporal encoder
   ->
latent dynamics or diffusion transformer
   ->
future video latent
   ->
decoder to pixels or features
```

现代 Video World Model 通常在 latent space 中生成，避免直接在 pixel space 做高分辨率时空建模。核心计算来自 3D/temporal attention、DiT 类 block、cross-attention conditioning 和多步 denoising 或 autoregressive rollout。

## 价值

Video World Model 的优势是信息完整，能保留外观、遮挡、动态、光照和复杂交互，因此适合做仿真可视化、数据生成、corner case replay 和人类审查。自动驾驶中的 GAIA-1、Waymo World Model，机器人和 physical AI 中的 Cosmos，都体现了这一方向。

问题是视频不是直接可规划状态。对 planner 来说，pixels 还要再被感知模型解释成 object、occupancy、risk 或 cost。生成视频如果几何略微漂移，人眼可能能接受，但 collision metric 不能接受。

## 成熟度判断

- 成熟：短视频生成、条件视频生成、离线数据扩增。
- 发展中：action-conditioned video world model、驾驶/机器人仿真可控生成。
- 前沿：可长期交互、可用于闭环 planning 的 high-fidelity video world model。

## 一句话理解

Video World Model 提供最直观的未来模拟，但它把 workload 推向高分辨率时空生成和多步 denoising，部署成本远高于结构化 latent/BEV 表示。

## Workload Characterization

- 计算密度：DiT/temporal attention 和 diffusion denoise 计算量高；高分辨率、多帧、多 sample 会成倍放大。
- 访存模式：video latent 是 5D 张量，时间、空间、channel 同时扩展；KV cache 或 activation 容量压力大。
- 并行性：不同 sample、不同 future candidate、不同 batch 可并行；单个 denoise chain 或 autoregressive horizon 存在串行步骤。
- 数据复用：history frame encoding 可复用于多个 future sample；map/action/text conditioning 可跨 rollout 复用。
- 量化敏感度：生成主干可混合精度；几何边界、交通灯、接触点和稀有小物体生成对误差敏感。
- 瓶颈类型：云端训练/生成受显存和吞吐限制；端侧实时视频 world model 目前成本很高。
- 对硬件的核心需求：高吞吐矩阵计算、大容量 activation/KV cache、视频 latent tiling、多样本并发、生成模型调度。

## 参考来源

- Hu et al., `GAIA-1: A Generative World Model for Autonomous Driving`, arXiv:2309.17080，https://arxiv.org/abs/2309.17080，成熟度：自动驾驶视频 world model 代表，查证日期：2026-05-29。
- OpenAI, `Video generation models as world simulators`, 2024，https://openai.com/index/video-generation-models-as-world-simulators/，成熟度：前沿系统展示，查证日期：2026-05-29。
- Google DeepMind, `Genie 2: A large-scale foundation world model`, 2024，https://deepmind.google/discover/blog/genie-2-a-large-scale-foundation-world-model/，成熟度：交互式 world model 前沿，查证日期：2026-05-29。
- Waymo, `The Waymo World Model`, 2026-02-06，https://waymo.com/blog/2026/02/the-waymo-world-model-a-new-frontier-for-autonomous-driving-simulation，成熟度：2026 产业仿真前沿，查证日期：2026-05-29。

## 旧版素材

- `/mnt/e/workload-wiki-old/05_World_Model与生成式智能/Video_World_Model.md`
