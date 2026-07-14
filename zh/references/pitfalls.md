# Profiling Pitfalls

本文只记录 workflow 层面的坑，关注证据边界，不讲 CUDA 或模型理论课程。

## Shape Drift

问题：

- 把 shortened/debug shape 误当作 official profiling shape。

规则：

- shape 是一等 artifact：input tensors、backend bindings、sequence length、action chunk、batch、precision。
- shape 变了，之前的 timing 和 NCU 结果不能自动比较。

## Framework Timing vs Semantic Timing

问题：

- 手动 semantic stage timing 可能绕过 optimized/compiled framework path。

规则：

- semantic timing 用于理解 stage 顺序和粗略占比。
- absolute latency 必须使用 official execution path。

## Backend Split Artifacts

问题：

- split engine 或 profiling surrogate 可能不等同 production engine。

规则：

- 如果用 split backend 做归因，必须和 single/backend baseline 对账。
- 记录差异，并判断它是否可接受。

## NVTX Range Misuse

问题：

- CPU-side NVTX range 常常包住 enqueue，而不是 GPU completion。

规则：

- 用 NSYS timeline 和 CUDA event timing 验证 NVTX range，再用于 NCU filtering。
- 如果 range 没覆盖所有相关 GPU kernels，不能作为最终 profiling boundary。

## Measurement Window Pollution

问题：

- setup、warmup、allocation、memset、H2D/D2H/D2D copy 或 sync 混入 measured window。

规则：

- 先用 NSYS 验证 clean window。
- setup/warmup 和 measured runs 分离。

## NCU Replay Time

问题：

- NCU replay 会改变执行，可能放大 kernel duration。

规则：

- 用 NSYS/CUDA event 做真实时间。
- 用 NCU 看 counters。
- 组合两者时，说明 join/alignment 方法。

## Representative Kernel Sampling

问题：

- 把少量 sampled kernels 当作 whole-stage result。

规则：

- representative samples 可以解释机制。
- stage-level 结论需要覆盖率：all kernels、all important families，或明确 sampling plan。

## Layer-to-Kernel Attribution

问题：

- 假设 runtime/compiler layer 和 CUDA kernel 一一对应。

规则：

- layer profile 只是语义提示。
- runtime 证据要看 NSYS timeline 和 kernel families。
- 说明 attribution uncertainty。

## Profiler Shape vs Model Shape

问题：

- 把 kernel launch grid、block size 或 CUTLASS tile string 当作完整模型 `M/N/K`。

规则：

- 模型 problem shape 来自 theory、graph、layer metadata 或 shape tracing。
- launch/tile information 用于一致性检查，不能替代理论 shape。

## Averages Hide Structure

问题：

- 平均值掩盖 step pattern、drift 或 rare outlier。

规则：

- 当行为可能周期性或漂移时，用 per-kernel / per-step visualization。
- 再决定报告用 statistic、timeline 还是代表性例子。

