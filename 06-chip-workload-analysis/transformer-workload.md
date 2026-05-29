# Transformer Workload

上级：[Chip Workload Analysis](README.md)
相关：[Attention and Transformer](../01-foundation-model-components/attention-and-transformer.md), [Attention Variants](../01-foundation-model-components/attention-variants-and-efficiency.md)

## 这页在回答什么问题

这页把 Transformer 拆成可建模 workload。Transformer 不能只写成 attention + FFN，因为 prefill、decode、vision encoder、cross-modal fusion 和 action token decode 的瓶颈不同。架构探索必须先区分阶段。

## Stage 拆解

| Stage | 典型场景 | 主导成本 |
| --- | --- | --- |
| prefill / encoder | ViT、VLM input、LLM prompt | 大 GEMM，compute-bound |
| attention score | QK^T、softmax、AV | 长序列时 memory 和 softmax 稳定性 |
| FFN / MLP | Transformer block 中最大 FLOPs 部分 | compute-bound，适合低比特 GEMM |
| decode | LLM/VLA action token 逐 token | KV cache bandwidth + latency-bound |
| cross attention | visual-language fusion | token 对齐和 KV 访问 |
| norm / reshape | RMSNorm、LayerNorm、transpose | 小算子搬运和同步开销 |

prefill 和 decode 是两个完全不同的 workload。prefill 可在 token 和 batch 维度并行，GEMM 规模大，MAC 利用率高；decode 每次只生成少量 token，反复读权重和 KV cache，常被 bandwidth 和 latency 限制。

## 关键参数

| 参数 | 放大什么 |
| --- | --- |
| `sequence length` | attention score、KV cache、softmax IO |
| `hidden size` | QKV/FFN GEMM FLOPs 和 weight size |
| `layer count` | 状态访问和 latency 线性放大 |
| `head count / head dim` | attention 并行、KV layout |
| `batch` | weight reuse；端侧 batch=1 最困难 |
| `KV precision` | cache 带宽和容量 |
| `prefill/decode ratio` | compute-bound 与 bandwidth-bound 的比例 |

## 硬件连接

- RAM：prefill 需要大块连续 weight/activation；decode 需要高效 KV cache layout 和带宽。
- DMA：prefill 适合 burst + double buffer；decode 需要小块、重复、低延迟搬运。
- NOC：多 tile attention 需要 Q/K/V 分发、softmax reduction、KV cache 共享路径。
- CIM：FFN、QKV projection、output projection 适合 CIM/GEMM 加速；softmax、KV cache、动态 decode 不是 CIM 主收益点。
- PCIE/host：云端模型跨设备时 PCIE/NVLink boundary 重要；端侧应避免 token decode 中 host 往返。

## archax 建模

- Resource：GEMM TOPS、SRAM/KV cache 容量、DRAM bandwidth、NoC reduction bandwidth。
- Topology：attention head 到 tile 的映射、KV cache 所在 memory level、host-device 边界。
- Interaction：`prefill blocks`、`decode loop`、`KV cache update`、`cross-modal token fusion`。
- Capability：GEMM、batched attention、online softmax、KV cache quantization、INT8/FP8/INT4、stateful decode。

## Workload Characterization

- 计算密度：prefill/encoder 高，通常 compute-bound；decode 低，通常 memory-bandwidth/latency-bound。
- 访存模式：GEMM 连续友好；KV cache 是状态访问；transpose/norm/softmax 会造成额外搬运。
- 并行性：prefill 可 token/head/batch 并行；decode 受 autoregressive 依赖限制，只能在 layer/head/GEMM 内并行。
- 数据复用：prefill weight/activation 复用好；decode batch 小，weight reuse 差，KV cache 成为主状态。
- 量化敏感度：FFN/QKV 可低比特；attention score、softmax、LayerNorm、KV cache 需要分 stage 验证。
- 瓶颈类型：prefill compute-bound；decode memory-bandwidth + latency-bound；长上下文 capacity-bound。
- 硬件核心需求：高效 GEMM、online attention、KV cache 布局、低 batch decode、低比特权重和 cache 支持。

## 参考来源

- Vaswani et al., `Attention Is All You Need`, NeurIPS 2017。
- Dao et al., `FlashAttention`, NeurIPS 2022。
- Pope et al., `Efficiently Scaling Transformer Inference`, MLSys 2023。
- `/mnt/e/workload-wiki-old/06_芯片架构Workload分析/Transformer_Workload.md`。
