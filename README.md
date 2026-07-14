# Profiling Workflow Skill

`profiling-workflow` is a Codex skill for top-down NVIDIA GPU profiling.

It is designed for inference or training workloads where the diagnosis must stay evidence-driven across:

```text
theoretical workload
  -> framework execution
  -> optional compiler/runtime backend
  -> Nsight Systems time structure
  -> conditional Nsight Compute kernel analysis
  -> visualization-assisted pattern extraction
  -> theory-measured alignment
  -> bounded conclusion
```

The skill is not an NSYS/NCU command collection and not a CUDA/GEMM tutorial. It defines the profiling workflow, evidence boundaries, required artifacts, metric groups, common pitfalls, and report style.

## Contents

```text
.
├── SKILL.md
├── references/
│   ├── evidence_matrix.md
│   ├── maintenance.md
│   ├── pitfalls.md
│   ├── report_style.md
│   └── workflow.md
└── zh/
    ├── SKILL.md
    └── references/
        ├── evidence_matrix.md
        ├── maintenance.md
        ├── pitfalls.md
        ├── report_style.md
        └── workflow.md
```

The root skill is the English installable version. The `zh/` directory is a Chinese copy for review and maintenance.

## Install Locally

Copy the repository contents to your Codex skills directory:

```bash
mkdir -p ~/.codex/skills/profiling-workflow
cp -a SKILL.md references ~/.codex/skills/profiling-workflow/
```

After installation, use the skill when planning or reviewing GPU profiling work, interpreting NSYS/NCU evidence, or writing profiling reports.

## Design Rules

- Start from theoretical workload, not profiler output.
- Run a no-profiler baseline before collecting traces.
- Treat compiler/runtime backends as conditional.
- Use NSYS to locate time and validate boundaries.
- Use NCU only when GPU kernel work needs explanation.
- Separate real timing from NCU replay counters.
- Keep project-specific scripts, paths, hardware values, and raw results outside the skill.

See `references/maintenance.md` before updating the skill based on a new profiling project.

