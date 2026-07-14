# Evidence Matrix

用这张表判断每一层必须证明什么。

| 层 | 获取什么 | 证明什么 | 排除什么 |
|---|---|---|---|
| Theory | stage、shape、`M/N/K`、FLOP/MAC、traffic、roofline、repetition structure | 预期 workload 和理论下限 | 只靠 profiler 输出编故事 |
| Framework | correctness、dtype/device/shape、baseline timing、semantic stage timing | 模型/输入有效，stage 顺序合理 | 错模型/输入、CPU fallback、不稳定 setup |
| Compiler/runtime | backend artifact、profile shape、precision、layer/operator profile、latency reconciliation | backend 与 workload 匹配 | 错 engine、错 profile、stage split artifact |
| NSYS | stage duration、CUDA API time、kernel time、copy/sync/allocation、gap、stream、range coverage | 时间花在哪里 | transfer/sync/setup/NVTX 污染 |
| NCU stage | trusted timing、kernel count、DRAM/L2、SM/memory throughput、occupancy | 哪个 stage 行为异常 | 证据不足的纯带宽或纯 launch 结论 |
| NCU family | family time share、family utilization、traffic、coverage | 哪些 kernel 重要 | 全局平均值结论 |
| NCU micro-arch | grid/block/waves、resources、TC/tensor/TMA pipes、eligible/issue、stalls、bank/branch/barrier | 重要 kernel 为什么慢 | 无证据根因 |
| Visualization | per-step/family/metric timeline | 稳定模式 vs 漂移/异常 | 过拟合单个聚合数字 |
| Alignment | theory vs framework/backend vs NSYS vs NCU | 最终因果故事 | 无证据 layer/kernel 一一对应 |

## Required vs Auxiliary

必需节点：

- theory；
- framework correctness；
- NSYS time structure；
- final theory-measured alignment。

条件节点：

- compiler/runtime alignment：当部署使用这类后端时；
- NCU family/stage behavior：当 NSYS 显示 GPU kernel work 需要解释时；
- micro-architecture：当 family/stage metrics 不足以解释时；
- visualization：当 repetition、drift 或 per-kernel 行为重要时。

辅助证据只有在能改变或加强结论时才有价值。

## 常见决策模式

| 观察 | 下一步问题 |
|---|---|
| NSYS 显示大 GPU idle gap | gap 来自 CPU enqueue、sync、dependency 还是缺少 overlap？ |
| NSYS 显示 copy/sync/allocation 主导 | 停留在 runtime/data-movement 层；NCU 不是下一步必需项。 |
| NSYS 没有 transfer/sync 问题，但 stage 仍慢 | 用 NCU 解释 kernel 行为。 |
| NCU 显示 SM 和 memory throughput 都低 | 先查 grid/waves 和 eligible warps，不要直接说 compute/memory bound。 |
| NCU 显示 waves/SM 低 | 查 problem shape、grid/block，以及 small `M/N` 是否解释它。 |
| waves/SM 高但 eligible warps 低 | 查 stall、memory dependency、barrier 和 source-level 行为。 |
| Tensor Core family 但 TC pipe active 低 | 可能用了 Tensor Core，但没把 pipe 喂满。 |
| Visualization 显示 per-step 重复 | 优先考虑结构性解释，而不是一次性异常。 |
| Theory 和 NCU 对不上 | 重新检查 shape、backend fusion/tactic、measurement window 和映射假设。 |
