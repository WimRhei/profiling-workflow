# Evidence Matrix

Use this table to decide what each profiling layer must prove.

| Layer            | Get                                                                | Proves                                          | Excludes                                              |
| ---------------- | ------------------------------------------------------------------ | ----------------------------------------------- | ----------------------------------------------------- |
| Theory           | stages, shapes, `M/N/K`, FLOP/MAC, traffic, roofline, repetition structure | expected workload and lower bound               | profiler-only speculation                             |
| Framework        | correctness, dtype/device/shape, baseline timing, semantic stage timing | model/input are valid; stage order is plausible | wrong model/input, CPU fallback, unstable setup       |
| Compiler/runtime | backend artifact, profile shapes, precision, layer/operator profile, latency reconciliation | backend matches workload                        | wrong engine, wrong profile, stage split artifact     |
| NSYS             | stage duration, CUDA API time, kernel time, copies/sync/allocation, gaps, streams, range coverage | where time is spent                             | transfer/sync/setup/NVTX pollution                    |
| NCU stage        | trusted timing, kernel count, DRAM/L2, SM/memory throughput, occupancy | which stage behavior is abnormal                | pure bandwidth or pure launch claims when unsupported |
| NCU family       | family time share, family utilization, traffic, coverage           | which kernels matter                            | global-average conclusions                            |
| NCU micro-arch   | grid/block/waves, resources, TC/tensor/TMA pipes, eligible/issue, stalls, bank/branch/barrier | why important kernels are slow                  | unsupported root causes                               |
| Visualization    | per-step/family/metric timeline                                    | stable pattern vs drift/anomaly                 | overfitting one aggregate number                      |
| Alignment        | theory vs framework/backend vs NSYS vs NCU                         | final causal story                              | layer/kernel one-to-one claims without evidence       |

## Required vs Auxiliary

Required nodes:

- theory;
- framework correctness;
- NSYS time structure;
- final theory-measured alignment.

Conditional nodes:

- compiler/runtime alignment, when deployment uses such a backend;
- NCU family/stage behavior, when NSYS shows GPU kernel work needs explanation;
- micro-architecture, when family/stage metrics are not enough;
- visualization, when repetition, drift, or per-kernel behavior matters.

Auxiliary evidence is useful only when it changes or strengthens the conclusion.

## Common Decision Patterns

| Observation | Next question |
|---|---|
| NSYS shows large GPU idle gaps | Are gaps caused by CPU enqueue, sync, dependencies, or missing overlap? |
| NSYS shows copy/sync/allocation dominates | Stay at runtime/data-movement level; NCU is not the next required step. |
| NSYS shows no transfer/sync issue but stage is slow | Use NCU to explain kernel behavior. |
| NCU shows low SM and low memory throughput | Check grid/waves and eligible warps before claiming compute or memory bound. |
| NCU shows low waves/SM | Check problem shape, grid/block, and whether small `M/N` explains it. |
| waves/SM high but eligible warps low | Investigate stalls, memory dependencies, barriers, and source-level behavior. |
| Tensor Core family but low TC pipe active | It may use Tensor Core but fail to keep the pipe busy. |
| Visualization shows repeated per-step pattern | Prefer structural explanation over one-off anomaly. |
| Theory and NCU disagree | Recheck shape, backend fusion/tactic, measurement window, and mapping assumptions. |
