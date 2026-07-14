# Profiling Report Style

报告应该像论证，不是数据 dump。

## 结构

按这个顺序写：

1. Purpose：本报告回答什么对齐问题或瓶颈问题。
2. Conclusion：最短的、证据支持的结论。
3. Evidence：只保留支撑结论所需的表格或图。
4. Boundary：说明证据不能证明什么。
5. Raw data：路径、命令、版本、配置。

## 最小骨架

```markdown
## Purpose
本报告回答什么对齐或瓶颈问题。

## Conclusion
一个被证据支持的结论，并用一句话说明排除了什么。

## Evidence
支撑结论所需的最小表格/图。

## Boundary
本报告不能证明什么；timing/counter/coverage caveats。

## Raw Data
report、script、config 路径和准确 measurement scope。
```

## 好结论的形状

使用：

```text
瓶颈是 X，由 A/B/C 支持。不是 Y/Z，因为 D/E。
```

避免：

```text
这里是所有 profiler metrics。
```

## 数据呈现

用统计值，当：

- 总结大量 kernels、stages 或 runs；
- 比较 families；
- 报告 launch shape 或 utilization 的 p50/p95/mean。

用 timeline visualization，当：

- 行为可能按 step/layer 重复；
- drift 或 periodicity 重要；
- 平均值掩盖结构；
- 用户需要检查 per-kernel metric 变化。

用单个例子，当：

- 解释支持结论的机制；
- 展示一个代表性 kernel shape；
- 验证 counter interpretation。

## Timing 规则

- 用 NSYS 或 CUDA event timing 作为真实 wall-clock time。
- NCU replay duration 是 profiling-time，不是正常运行时间。
- 如果把 NCU metrics join 到 NSYS timing，说明 join key 和限制。

## 边界语言

使用明确边界：

- “NCU raw 不提供完整 GEMM `M/N/K`；它提供 launch 和 tactic。”
- “TensorRT layer 和 CUDA kernel 不是一一对应。”
- “这证明数量/顺序一致性，不证明精确 layer-kernel mapping。”
- “这排除了 transfer/sync 是主因，但不排除所有 runtime overhead。”

## 最小最终检查

- 是否说明理论 workload？
- model/framework/backend shapes 是否对齐？
- measurement window 是否 clean？
- 真实 timing 和 NCU replay timing 是否分开？
- micro-arch 细节前，是否先识别重要 kernel families？
- counter interpretation 是否绑定到它实际能证明的内容？
- alternative explanations 是否明确排除或保留？

