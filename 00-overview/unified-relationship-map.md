# Unified Relationship Map

上级：[Overview](README.md)
相关：[Wiki Roadmap](wiki-roadmap.md), [Robotics vs Autonomous Driving](../04-robotics-and-vla/robotics-vs-autonomous-driving.md), [World Model Fundamentals](../05-world-model-and-generative/world-model-fundamentals.md)

## 这页在回答什么问题

这页把自动驾驶、机器人、大模型和 World Model 四条看似独立的线，画进一张统一关系图。它们之所以能放进同一份 wiki，是因为从 workload 视角看，它们共享同一批基础组件，只是组合方式和部署约束不同。看懂这张图，就知道为什么 06 能用一套维度统一刻画它们。

## 共享的底座

四条线都建在 01 章那批基础组件上：CNN/ViT 做视觉编码，Transformer/Attention 做 token 交互与多模态融合，Diffusion 做生成与多模态分布建模，Mamba/SSM 做长序列压缩，自监督（对比学习、JEPA）提供表征底座。换句话说，自动驾驶感知、机器人 policy、视频生成、World Model rollout，底层算子高度重叠——都是 GEMM、attention、conv、迭代生成、状态递推的不同配比。这就是它们能共用一套 Workload Characterization 维度的根本原因。

## 四条线如何分化又如何交汇

```text
                 基础组件 (01: CNN/ViT/Transformer/Diffusion/Mamba/SSL)
                                |
        +-----------------------+------------------------+
        |                       |                        |
   视觉与3D感知(02)         序列/生成建模              自监督表征
   检测/分割/LiDAR          (Diffusion/SSM)           (Contrastive/JEPA)
        |                       |                        |
     BEV / Occupancy  <---------+                        |
        |  (02->03 桥梁)                                  |
        v                                                 v
   自动驾驶(03)                                      World Model(05)
   modular->BEV->E2E                                 latent/video/BEV/occ
   ->VLM/VLA for AD                                  ->端云协同
        |                                                 |
        +------------------> 机器人 VLA(04) <-------------+
                       vision-language-action
                       action tokenizer / flow policy
                                |
                                v
                  芯片 Workload 分析(06, 重心)
        CNN / Transformer / BEV / Occupancy / E2E / VLA / World Model
                                |
                                v
              七份硬件 wiki (BUS/RAM/NOC/DMA/FAB/CIM/PCIE) + archax
```

感知（02）产出 BEV/Occupancy，它既是视觉感知的终点，又是自动驾驶（03）的起点，所以 BEV 放在 02 末尾作桥梁。自动驾驶的 E2E 和机器人的 VLA（04）在结构上正在收敛——两者都从模块化走向"感知到动作"的端到端大模型，都引入 VLM/VLA 范式，差异主要在部署约束（车规、控制频率、安全壳）而非算法骨架（见 [Robotics vs Autonomous Driving](../04-robotics-and-vla/robotics-vs-autonomous-driving.md)）。World Model（05）则横跨两侧：既给自动驾驶做仿真和安全评估，又给机器人做 latent dynamics 和规划，还和视频生成共享 Diffusion 组件但目标不同。

## 自动驾驶与机器人 VLA 的同与异

相同：都需要多模态感知、时序状态、端到端 policy、闭环低延迟、端云数据闭环；底层 workload 都是 dense compute + irregular access + stateful cache + bounded latency 的组合。

不同：自动驾驶的输入以环视多相机+LiDAR 为主、输出是轨迹、安全要求达车规级、控制频率相对低；机器人 VLA 输入含本体状态和腕部相机、输出是高频连续控制动作、embodiment 多样、跨平台泛化是核心难题。这些差异落到芯片上，是 KV cache/BEV cache 的形态、action 输出的 decode 方式、以及 safety 路径设计的不同。

## World Model 的横向角色

World Model 不是第五条独立赛道，而是连接其余三条的"想象未来"能力层：给自动驾驶当仿真器和安全评估器，给机器人当 latent dynamics 和规划引擎，借用 Diffusion/Transformer 的生成能力但追求可控状态预测而非像素逼真（见 [World Model Is Not Video Generation](../05-world-model-and-generative/world-model-is-not-video-generation.md)）。它的 workload 是一个 `候选 × 时域 × 单步` 的乘法，因此天然分裂为端侧轻量 latent rollout 和云端重型生成两端。

## 一句话理解

自动驾驶、机器人、大模型、World Model 不是四份并列内容，而是同一批基础组件在不同部署约束下的组合；这张图的终点都汇向 06，再接到七份硬件 wiki 和 archax。
