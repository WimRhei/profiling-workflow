# Profiling Report Style

Write reports as an argument, not as a dump.

## Structure

Use this order:

1. Purpose: what alignment or question this report answers.
2. Conclusion: the shortest supported claim.
3. Evidence: only the tables or figures needed for the claim.
4. Boundary: what the evidence does not prove.
5. Raw data: paths, commands, versions, configs.

## Minimal Skeleton

```markdown
## Purpose
What alignment or bottleneck question this report answers.

## Conclusion
One supported claim, plus one sentence naming what was excluded.

## Evidence
Smallest table/figure set needed for the claim.

## Boundary
What this report does not prove; timing/counter/coverage caveats.

## Raw Data
Paths to reports, scripts, configs, and exact measurement scope.
```

## Good Conclusion Shape

Use:

```text
The bottleneck is X, supported by A/B/C. It is not Y/Z because D/E.
```

Avoid:

```text
Here are all profiler metrics.
```

## Data Presentation

Use statistics when:

- summarizing many kernels, stages, or runs;
- comparing families;
- reporting p50/p95/mean for launch shape or utilization.

Use timeline visualization when:

- behavior may repeat by step/layer;
- drift or periodicity matters;
- averages hide structure;
- user needs to inspect per-kernel metric changes.

Use a single example when:

- explaining a mechanism that supports the conclusion;
- showing one representative kernel shape;
- verifying a counter interpretation.

## Timing Rules

- Use NSYS or CUDA event timing for real wall-clock time.
- Treat NCU replay duration as profiling-time, not normal runtime.
- If NCU metrics are joined to NSYS timing, state the join key and limitations.

## Boundary Language

Use explicit boundary statements:

- “NCU raw does not provide complete GEMM `M/N/K`; it provides launch and tactic.”
- “TensorRT layer and CUDA kernel are not one-to-one.”
- “This proves quantity/order consistency, not exact layer-kernel mapping.”
- “This excludes transfer/sync as primary cause, but not all runtime overhead.”

## Minimum Final Checklist

- Is the theoretical workload stated?
- Are model/framework/backend shapes aligned?
- Is the measurement window clean?
- Are real timings separated from NCU replay timings?
- Are important kernel families identified before micro-arch details?
- Are counter interpretations tied to what they can actually prove?
- Are alternative explanations explicitly excluded or left open?
