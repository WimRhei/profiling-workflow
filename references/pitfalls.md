# Profiling Pitfalls

This file records workflow-level traps. It should stay about evidence boundaries, not CUDA or model theory lessons.

## Shape Drift

Problem:

- A shortened or debug shape is accidentally used as the official profiling shape.

Rule:

- Treat shape as a first-class artifact: input tensors, backend bindings, sequence length, action chunk, batch, precision.
- If shape changes, earlier timing and NCU results are not automatically comparable.

## Framework Timing vs Semantic Timing

Problem:

- Manual semantic stage timing may bypass the optimized or compiled framework path.

Rule:

- Use semantic timing to understand ordering and rough contribution.
- Use the official execution path for absolute latency claims.

## Backend Split Artifacts

Problem:

- A split engine or profiling surrogate may not match the production engine.

Rule:

- If using a split backend for attribution, reconcile it against the single/backend baseline.
- Record the delta and decide whether it is acceptable for profiling.

## NVTX Range Misuse

Problem:

- CPU-side NVTX ranges often wrap enqueue, not GPU completion.

Rule:

- Validate NVTX ranges with NSYS timeline and CUDA event timing before using them for NCU filtering.
- If the range does not cover all relevant GPU kernels, do not use it as a final profiling boundary.

## Measurement Window Pollution

Problem:

- Setup, warmup, allocation, memset, H2D/D2H/D2D copy, or sync enters the measured window.

Rule:

- Use NSYS first to verify a clean window.
- Keep setup/warmup separate from measured runs.

## NCU Replay Time

Problem:

- NCU replay changes execution and can inflate kernel duration.

Rule:

- Use NSYS/CUDA events for real time.
- Use NCU for counters.
- When combining them, state the join/alignment method.

## Representative Kernel Sampling

Problem:

- A few sampled kernels are treated as a whole-stage result.

Rule:

- Representative samples can explain a mechanism.
- Stage-level conclusions need coverage: all kernels, all important families, or an explicit sampling plan.

## Layer-to-Kernel Attribution

Problem:

- Runtime/compiler layers are assumed to map one-to-one to CUDA kernels.

Rule:

- Treat layer profiles as semantic hints.
- Use NSYS timeline and kernel families for runtime evidence.
- State attribution uncertainty.

## Profiler Shape vs Model Shape

Problem:

- Kernel launch grid, block size, or CUTLASS tile strings are mistaken for complete model `M/N/K`.

Rule:

- Get model problem shape from theory, graph, layer metadata, or shape tracing.
- Use launch/tile information to check consistency, not to replace theory.

## Averages Hide Structure

Problem:

- Averages make repeated step patterns, drift, or rare outliers invisible.

Rule:

- Use per-kernel or per-step visualization when behavior may be periodic or drifting.
- Then decide whether the right presentation is a statistic, a timeline, or one representative example.

