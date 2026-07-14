# Maintaining This Profiling Skill

Use this guide after profiling a new project. The goal is to improve the skill without turning it into a project log, tool manual, or GPU knowledge textbook.

## What Belongs in the Skill

Add to `SKILL.md` when the change affects the global workflow:

- a new required or conditional stage;
- a new decision gate;
- a clearer minimum execution step;
- a new output expectation that should apply to most NVIDIA GPU profiling tasks.

Add to `references/workflow.md` when the change clarifies one stage:

- what to collect;
- why the stage exists;
- what the stage proves;
- what the stage excludes;
- required metric groups;
- output artifact shape.

Add to `references/evidence_matrix.md` when a new observation changes the diagnostic path:

- observation -> check -> next step;
- required vs conditional status;
- evidence that rules out a common wrong conclusion.

Add to `references/pitfalls.md` when a mistake is reusable across projects:

- profiling boundary misuse;
- shape/config drift;
- wrong timing source;
- partial coverage treated as full evidence;
- profiler field misinterpreted as model property;
- semantic layer attribution overstated as kernel attribution.

Add to `references/report_style.md` when a reporting rule improves conclusion quality:

- how to state evidence boundaries;
- how to choose statistic vs timeline vs representative example;
- how to separate timing, counters, coverage, and uncertainty.

## What Does Not Belong in the Skill

Do not add:

- project-specific scripts or commands;
- absolute paths;
- one project's model names, checkpoints, tasks, or dataset;
- hardware-specific values unless they are examples in a separate project reference;
- detailed CUDA, GEMM, Tensor Core, or NCU metric teaching material;
- raw experiment results;
- one-off debugging notes;
- outdated plans.

These should live elsewhere.

## Where Project Information Should Go

Use project documents for project-specific facts:

| Information | Put it in |
|---|---|
| Official model/backend/hardware/run configuration | project config document |
| Current profiling status and key numbers | project status document |
| Formal experiment result | results / experiment records |
| Raw reports, traces, CSVs, generated HTML | official raw data directory |
| Failed route that affects future decisions | project decisions document |
| Tool command details or platform-specific collection method | project reference document |
| Domain knowledge learned during the project | project reference document |
| Temporary smoke/debug attempts | archive or scratch area, not the skill |

## Abstraction Rule

Before adding anything to this skill, ask:

1. Would this apply to another NVIDIA GPU profiling project?
2. Does it change the diagnostic order, evidence boundary, or output artifact?
3. Is it independent of a specific script, model, path, hardware SKU, or run date?
4. Is it concise enough to guide future work without teaching a whole topic?

If the answer is no, keep it in the project.

## How to Distill a New Profiling Lesson

Use this pattern:

```text
Project observation:
  What happened in the specific project?

Wrong conclusion risk:
  What would someone incorrectly infer?

Reusable rule:
  What should future profiling workflows do differently?

Skill destination:
  SKILL.md / workflow.md / evidence_matrix.md / pitfalls.md / report_style.md / outside skill
```

Example:

```text
Project observation:
  CPU NVTX range covered enqueue but not all GPU kernels.

Wrong conclusion risk:
  Use NVTX include for NCU and miss relevant kernels.

Reusable rule:
  Validate NVTX range against NSYS kernel timestamps or CUDA events before using it as a profiling boundary.

Skill destination:
  pitfalls.md and workflow.md NSYS boundary stage.
```

## Periodic Review Checklist

After each major profiling project, review:

- Did the workflow order still work?
- Did any stage become required or conditional in a new way?
- Did any metric group become necessary for diagnosis?
- Was there a new pitfall that could recur?
- Did any report boundary language prevent a wrong conclusion?
- Did any skill content become too project-specific and need removal?

Keep the skill small. Prefer one reusable sentence over a project-specific paragraph.

