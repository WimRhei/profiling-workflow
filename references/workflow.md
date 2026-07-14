# Profiling Workflow

Use this order. Each layer should justify why the next layer is needed.

## 1. Theoretical Workload

Required.

Collect:

- model stages and dependencies;
- input shapes, batch, sequence length, action chunk, denoise/iteration count;
- GEMM/attention `M/N/K` where possible;
- theoretical FLOP, tensor traffic, and roofline lower bound;
- expected repeated structure across layers/steps.

Metric groups:

| Group | Answers |
|---|---|
| Shape | What problem is the model asking the GPU to solve? |
| FLOP / MAC | How much ideal compute exists? |
| Tensor traffic | How much ideal data movement exists? |
| Roofline lower bound | What is the optimistic latency floor? |
| Repetition structure | Should kernel behavior repeat by layer, step, token, or microbatch? |

Purpose:

- establish the workload before profiler noise;
- provide a reference for every later measurement;
- prevent interpreting launch/tactic data as model shape.

Output artifact:

| Field | Required content |
|---|---|
| Model / config | checkpoint, model variant, task/input profile |
| Stage map | stage names, dependencies, repeated loops/steps |
| Input shape | batch, sequence/action length, image/state/noise shapes as applicable |
| Precision | framework/backend dtype or quantization |
| Heavy ops | expected GEMM/attention/conv or other dominant ops |
| Theory numbers | FLOP/MAC, traffic, roofline lower bound when available |
| Source of truth | model code, graph export, backend layer metadata, or analysis script |

## 2. Framework Execution

Required.

Collect:

- correctness against reference output;
- device/dtype/batch/shape confirmation;
- semantic stage timing;
- warmup/run stability;
- power/clock/runtime settings when relevant.

Metric groups:

| Group | Answers |
|---|---|
| Correctness | Is this the right model/input/output? |
| Device / dtype / shape | Is execution on the intended accelerator and problem size? |
| Baseline latency | What is the no-profiler reference timing? |
| Warmup / variance | Is timing stable enough to profile? |
| Semantic stage timing | Which model region is plausibly important? |

Purpose:

- prove the model and input are correct;
- establish stage order and rough bottleneck;
- detect shape or configuration drift early.

Exclude:

- CPU fallback;
- wrong checkpoint/config/input;
- unstable warmup or clock state.

Output artifact:

| Field | Required content |
|---|---|
| Correctness | reference comparison or expected output shape/value check |
| Runtime config | device, dtype, batch/shape, warmup/runs |
| Baseline timing | latency mean/std or representative timing |
| Semantic stages | stage order and rough timing, if available |
| Limitation | whether timing is production path, compiled path, or semantic/debug path |

## 3. Compiler / Runtime Backend

Conditional. Use if the deployment path uses TensorRT, ONNX Runtime, Triton, XLA, TVM, CUDA Graph, or another compiler/runtime layer.

Collect:

- engine/model artifact, precision, binding shapes, dynamic shape profile;
- layer profile or operator profile;
- stage split and single-vs-split consistency if applicable;
- transfer policy and GPU-resident buffers;
- CUDA Graph or runtime enqueue behavior if relevant.

Metric groups:

| Group | Answers |
|---|---|
| Binding / operator shapes | Does backend shape match theory/framework? |
| Precision / quantization | Is the backend using the intended numeric path? |
| Layer/operator profile | Which semantic regions dominate inside the backend? |
| Backend latency | Does backend latency reconcile with baseline? |
| Transfer policy | Are inputs/intermediates staying where expected? |

Purpose:

- prove the profiled backend matches the theoretical workload;
- verify stage boundaries and precision;
- avoid profiling the wrong engine or shape.

Exclude:

- engine shape mismatch;
- unexpected host/device transfer;
- stage split overhead large enough to change the conclusion;
- layer/runtime profile inconsistent with theory.

Pitfall:

- A backend layer/operator is not necessarily a CUDA kernel. Use layer profiles for semantic attribution, not as a one-to-one kernel map.

Output artifact:

| Field | Required content |
|---|---|
| Backend artifact | engine/model/profile path or runtime configuration |
| Shape profile | bindings or operator input shapes |
| Precision | quantization/dtype by major stage if relevant |
| Baseline reconciliation | backend vs framework or production baseline delta |
| Layer/operator profile | top operators/layers when available |
| Boundary | what backend profile can and cannot prove |

## 4. NSYS Time Structure

Required before NCU.

Collect:

- measured window boundaries;
- stage duration and CUDA event consistency;
- CUDA memcpy/memset/allocation/synchronization in the measured window;
- kernel launch density and API enqueue time;
- GPU idle gaps and stream overlap;
- NVTX range coverage.

Metric groups:

| Group | Answers |
|---|---|
| Stage duration | Which region consumes wall-clock time? |
| CUDA API enqueue time | Is CPU/runtime submission expensive? |
| Kernel duration sum | How much time is actual GPU kernel work? |
| GPU idle / gap | Is the GPU waiting between kernels or stages? |
| Stream overlap / concurrency | Is available stream parallelism being used? |
| Memcpy / memset / allocation / sync | Is the measured window polluted or data-movement/sync dominated? |
| NVTX / range coverage | Is the profiling boundary trustworthy? |

Purpose:

- answer where time is spent;
- determine whether launch/gap/transfer/sync dominates;
- validate boundaries before collecting hardware counters.

Exclude:

- setup/warmup contamination;
- unexpected H2D/D2H/D2D copies;
- NVTX ranges that do not cover GPU work;
- CPU enqueue or sync as the primary bottleneck when evidence says otherwise.

Pitfall:

- CPU-side NVTX ranges may cover enqueue time, not GPU completion. Validate against CUDA events or kernel timestamps before using NVTX for NCU filtering.

Output artifact:

| Field | Required content |
|---|---|
| Window | measured range, warmup/setup separation |
| Stage timing | CUDA event or trusted wall-clock timing |
| Timeline checks | copies, memset, allocation, sync, stream behavior |
| Launch/gap | launch density/API time, GPU idle/gap, overlap |
| Boundary verdict | whether the window is valid for deeper profiling |
| Next decision | runtime/transfer/sync investigation or NCU |

## 5. NCU Stage and Family Analysis

Conditional. Required only when NSYS shows GPU kernel work needs explanation.

Collect:

- stage-level kernel count, latency from trusted timing, DRAM/L2 traffic, SM throughput, memory throughput, occupancy;
- kernel family time attribution using NSYS timing where NCU replay would distort duration;
- family-level utilization and traffic.

Metric groups:

| Group | Answers |
|---|---|
| Trusted timing | How much real time does this stage/family consume? |
| Kernel count | Is work fragmented into many kernels? |
| DRAM read/write or bytes | How much external memory traffic exists? |
| L2 traffic / requests | How much cache-level traffic exists? |
| SM throughput | Are SM execution resources busy? |
| Memory throughput | Is the memory system busy? |
| Occupancy | Are enough warps resident to hide latency? |
| Family time share | Which kernel families deserve deeper analysis? |

Purpose:

- identify which stage and kernel families matter;
- avoid drawing conclusions from global averages;
- separate compute, memory, low-util, and auxiliary work.

Exclude:

- pure memory bandwidth limit if DRAM bandwidth is low;
- irrelevant small families;
- NCU replay time as real wall-clock time.

Pitfall:

- Representative kernel samples are smoke/debug evidence. Do not use them as stage-level DRAM, utilization, or top-family conclusions unless coverage is proven.

Output artifact:

| Field | Required content |
|---|---|
| Coverage | stage/window/kernels covered, filtering method |
| Stage summary | kernel count, trusted latency, DRAM/L2, SM/memory throughput, occupancy |
| Family summary | top families by trusted time, utilization, traffic |
| Timing source | NSYS/CUDA event timing vs NCU replay counters |
| Verdict | compute, memory, low-util, launch geometry, or auxiliary work |

## 6. NCU Per-Kernel / Micro-Architecture

Use after family analysis identifies important kernels.

Collect:

- grid size, block size, waves/SM;
- SM throughput, TC pipe active, tensor pipe active, TMA pipe active;
- occupancy and active/eligible warps;
- issue active and stall breakdown;
- barrier, branch divergence, shared-memory bank conflict indicators;
- source/SASS evidence when needed to prove Tensor Core or instruction path.

Metric groups:

| Group | Answers |
|---|---|
| Grid / block / waves | Does one launch expose enough GPU-level parallelism? |
| Registers / shared memory | Are per-block resources limiting occupancy or tactic choice? |
| SM / TC / tensor / TMA pipe | Which execution path is active and whether it is saturated? |
| Active / eligible warps | Are schedulers supplied with ready work? |
| Issue active | Are schedulers actually issuing instructions? |
| Stall breakdown | What prevents ready instruction issue? |
| Long scoreboard | Are long-latency dependencies dominant? |
| Barrier stall | Is synchronization limiting progress? |
| Shared-memory conflict | Is shared-memory access inefficient? |
| Branch divergence | Is warp control-flow divergence relevant? |

Purpose:

- distinguish GPU-level parallelism from active-SM micro-architecture problems;
- detect supply, dependency, barrier, branch, or shared-memory issues;
- prove whether Tensor Core is used and whether it is busy.

Exclude:

- “not using Tensor Core” when SASS/family counters show Tensor Core use;
- bank conflict, branch divergence, or barrier as root cause when counters do not support it;
- small-grid as sole cause when waves are high but eligible warps remain low.

Pitfall:

- Kernel names and launch fields do not provide complete GEMM `M/N/K`. CUTLASS tile strings describe tactic/tile shape, not the full model problem.

Output artifact:

| Field | Required content |
|---|---|
| Target kernels | family/shape/sample coverage and why selected |
| GPU-level | grid, block, waves/SM, occupancy |
| Compute path | SM throughput, TC/tensor/TMA pipe, SASS/source if needed |
| Scheduler | eligible warps, issue active, active warps |
| Stall/conflict | dominant stalls, barrier, branch, shared-memory conflict |
| Exclusions | which suspected causes are ruled out |

## 7. Visualization-Assisted Pattern Extraction

Auxiliary. Use when kernel behavior is repeated, step-based, drifting, or difficult to summarize with aggregate statistics.

Use:

- NSYS GUI for timeline sanity and stream behavior;
- independent HTML/CSV timeline when combining NSYS time with NCU metrics;
- stage/step/family/metric filters;
- per-kernel bar/timeline views for SM, TC pipe, memory, waves, or stalls.

Purpose:

- tell whether behavior is stable, periodic, drifting, or localized;
- separate structural repetition from one-off anomalies;
- guide which statistics should be reported.

Do not use visualization alone as proof of a hardware bottleneck. Pair it with counters and theory.

Output artifact:

| Field | Required content |
|---|---|
| View | stage/step/family/metric filters used |
| Pattern | repeated, drifting, localized, or outlier behavior |
| Metric choice | which metric best exposes the pattern |
| Reporting choice | statistic, timeline, or representative example |

## 8. Theory-Measured Alignment

Required before final conclusion.

Align:

- theoretical stage workload vs measured stage timing;
- theoretical `M/N/K`, FLOP, traffic vs NCU family/kernel behavior;
- backend layer/operator profile vs CUDA kernel families;
- NSYS wall time vs NCU counters.

Purpose:

- decide whether measurements are explained by model problem shape, runtime overhead, memory behavior, or micro-architecture;
- state evidence boundaries clearly.

If layers do not align, investigate shape, stage boundary, profiling window, compiler fusion, tactic changes, or unsupported theory assumptions before finalizing.

Output artifact:

| Layer | Alignment check |
|---|---|
| Theory | expected stages, shapes, FLOP/traffic |
| Framework | correctness, actual shapes, semantic timing |
| Backend | engine/operator shape and latency reconciliation, if used |
| NSYS | real timing, clean window, launch/gap/stream behavior |
| NCU | family/counter evidence, coverage, timing caveat |
| Conclusion | supported claim, exclusions, remaining uncertainty |
