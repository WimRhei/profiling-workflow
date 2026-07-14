---
name: profiling-workflow
description: Use this skill for NVIDIA GPU profiling workflows that need a top-down, evidence-driven diagnosis across model theory, framework execution, optional compiler/runtime backends such as TensorRT, Nsight Systems, Nsight Compute, and theory-vs-measured alignment. Use it when analyzing PyTorch/CUDA/TensorRT/NVIDIA GPU performance for inference or training, building profiling plans, interpreting NSYS/NCU results, or writing profiling reports with clear evidence boundaries.
---

# Profiling-Workflow

Use this skill to diagnose NVIDIA GPU performance for inference or training. The goal is not to run every profiler immediately; the goal is to build a self-consistent evidence chain.

Core workflow:

```text
1. Build the theoretical workload expectation.
2. Run the model once without profilers and verify correctness, shape, dtype, device, and baseline latency.
3. If a compiler/runtime backend is used, reconcile its shape, precision, stages, and latency with the baseline.
4. Use NSYS to locate where time is spent: CPU/runtime, copy/sync/gap, launch, stream behavior, or GPU kernels.
5. Use NCU only when GPU kernel work needs explanation.
6. Compare theory, framework/backend, NSYS, and NCU evidence before making a bounded conclusion.
```

## Required Discipline

- Start from theory: model stages, tensor shapes, batch, sequence/action length, GEMM/attention `M/N/K`, FLOP, traffic, roofline lower bound.
- At every deeper layer, check whether measured behavior matches the theory. If not, suspect shape, stage boundary, warmup/run, backend conversion, or measurement window before making a performance claim.
- Treat compiler/runtime backends as conditional. If TensorRT/ONNX Runtime/Triton/XLA exists, align it; if not, skip that layer.
- Do not bake project-specific scripts into the workflow. Use project scripts only as local implementations of the same evidence requirements.

## When Starting

Read [references/workflow.md](references/workflow.md) before designing or executing a profiling plan.

Read [references/evidence_matrix.md](references/evidence_matrix.md) when deciding what each stage must prove or exclude.

Read [references/report_style.md](references/report_style.md) before writing a profiling report.

Read [references/pitfalls.md](references/pitfalls.md) when a conclusion depends on NVTX ranges, backend shape, NCU replay duration, representative kernel sampling, layer-kernel attribution, or profiler-derived GEMM shape.

Read [references/maintenance.md](references/maintenance.md) before updating this skill based on a new profiling project.

## Minimum Execution Checklist

1. Build the theoretical workload artifact: stages, shapes, dtype, batch/sequence/action length, expected heavy ops, FLOP/traffic if available.
2. Run once without profiler and record correctness, device, dtype, shape, warmup/run, and baseline latency.
3. If a compiler/runtime backend is used, verify backend shape/profile/precision and reconcile it with the framework or production baseline.
4. Run a short NSYS trace to validate measurement window, stage boundaries, copies, sync, gaps, stream behavior, and launch density.
5. Decide from NSYS:
   - if CPU enqueue, copy, sync, allocation, or idle gap explains the issue, stay at NSYS/runtime level;
   - if GPU kernel work needs explanation, collect NCU.
6. In NCU, start with stage/family summaries before per-kernel or micro-architecture metrics. At minimum cover timing source, traffic, throughput, occupancy, launch geometry, scheduler readiness, stall reasons, and relevant compute pipes.
7. Use visualization when behavior may repeat, drift, or hide behind averages.
8. Write an alignment table: theory vs framework/backend vs NSYS vs NCU.
9. State the conclusion, what was excluded, and what remains unproven.

## Decision Tree

```text
Start with theory + no-profiler baseline.

If correctness, shape, dtype, device, or baseline is wrong:
  fix the run before profiling.

If a compiler/runtime backend is used and does not match the framework/baseline:
  fix backend shape, precision, stage split, or runtime configuration.

Run short NSYS.

If time is explained by CPU enqueue, copy, sync, allocation, or idle gap:
  stay at runtime / data movement / scheduling analysis.

If the measured window is polluted or stage boundaries are unreliable:
  fix the profiling boundary before using NCU.

If GPU kernel work dominates and remains unexplained:
  collect NCU stage/family data.

If one or two kernel families dominate:
  inspect per-kernel and micro-architecture metrics for those families only.

If behavior repeats by step/layer or averages hide structure:
  use timeline visualization to decide whether to report statistics, a timeline, or representative examples.

Before final conclusion:
  align theory, framework/backend, NSYS timing, and NCU counters.
```

## Output Expectations

A good profiling answer should state:

- what theoretical workload is expected;
- what each measurement layer proves;
- what each layer rules out;
- where the evidence does not support a stronger claim;
- whether the conclusion is CPU/runtime overhead, transfer/sync/gap, GPU kernel work, GPU-level parallelism, memory behavior, micro-architecture, or model problem shape.
