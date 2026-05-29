# Workload Comparison Table

上级：[Reference](README.md)
相关：[Workload Characterization Axes](../06-chip-workload-analysis/workload-characterization-axes.md), [Chip Workload Analysis](../06-chip-workload-analysis/README.md)

## 这页在回答什么问题

这页把主要 workload 放到同一张对比表中。它服务架构探索：先看瓶颈和硬件需求，再决定应该深入哪篇 06 文档。

## 总表

| Workload | 计算密度 | 访存模式 | 第一瓶颈 | 数据复用 | 量化敏感度 | 硬件重点 |
| --- | --- | --- | --- | --- | --- | --- |
| CNN standard conv | 高 | 规则 strided | compute | weight/activation 强 | INT8 成熟 | Conv/GEMM array、SRAM tiling |
| Depthwise / mobile CNN | 低到中 | 规则但复用弱 | bandwidth/latency | activation 有限 | small channel 需验证 | DMA、fusion、低开销调度 |
| Transformer prefill | 高 | 连续 GEMM + attention | compute | weight/activation 强 | FFN/QKV 可低比特 | GEMM、online attention |
| Transformer decode | 低 | KV cache 状态访问 | bandwidth/latency | KV reuse 但读写重 | KV/cache 需验证 | cache layout、低 batch decode |
| BEV | 中 | gather/scatter + state cache | irregular access | BEV feature 强 | geometry 需保守 | scatter-gather DMA、NoC、BEV cache |
| Occupancy | 中 | dense 3D 或 sparse irregular | capacity / irregular | history/map 可复用 | boundary 敏感 | 3D tiling、sparse metadata |
| E2E AD | 混合 | 多 stage + state + sync | p99 latency | shared BEV/query | trajectory/safety 敏感 | deterministic pipeline |
| VLA | 混合 | visual token + KV + action state | memory/latency | context/action chunk 可复用 | action head 敏感 | low-batch VLM、KV cache |
| World Model | 取决于表示 | rollout state + generated latent | rollout cost/capacity | condition/state 强 | long horizon 敏感 | candidate parallel、state cache |
| Cloud simulation | 高但 IO 重 | batch tensor + data lake | IO/communication/throughput | model weights / scenario | metric 需一致 | HBM、interconnect、storage IO |

## 端侧优先级

端侧优先关注 p99 latency、状态缓存和数据搬运。适合优先优化：

- BEV view transform 的 scatter-gather 和 BEV cache。
- Transformer/VLA decode 的 KV cache 和 batch=1 latency。
- E2E pipeline 的 CPU/NPU 同步和 safety shell。
- Occupancy 的 3D grid 容量和 sparse metadata。

## 云端优先级

云端优先关注吞吐、容量、通信和数据管线。适合优先优化：

- VLM/World Model 大 batch prefill 和生成吞吐。
- 仿真 scenario 并发和多版本模型回放。
- video/occupancy world model 训练显存和 HBM。
- 数据湖读取、解码、写回和调度。

## Workload Characterization

本页是对比表，不代表单一 workload。它把 06 的统一维度压缩成可快速扫描的矩阵，用于选择下一步架构建模入口。
