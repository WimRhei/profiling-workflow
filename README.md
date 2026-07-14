# Profiling Workflow Skill

<p>
  <a href="#中文"><img alt="中文" src="https://img.shields.io/badge/README-中文-blue"></a>
  <a href="#english"><img alt="English" src="https://img.shields.io/badge/README-English-gray"></a>
</p>

## 中文

`profiling-workflow` 是一个 Codex skill，用于自顶向下地分析 NVIDIA GPU profiling 结果。

它面向推理或训练 workload，核心要求是让诊断始终围绕证据展开：

```text
理论 workload
  -> 框架执行
  -> 可选的编译器/运行时后端
  -> Nsight Systems 时间结构
  -> 条件触发的 Nsight Compute kernel 分析
  -> 借助可视化提取模式
  -> 理论与实测对齐
  -> 有边界的结论
```

这个 skill 不是 NSYS/NCU 命令合集，也不是 CUDA/GEMM 教程。它定义的是 profiling 工作流、证据边界、必需产物、指标组、常见陷阱和报告风格。

### 内容

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

根目录是英文可安装版本。`zh/` 目录是中文副本，用于审阅和维护。

### 本地安装

把仓库内容复制到 Codex skills 目录：

```bash
mkdir -p ~/.codex/skills/profiling-workflow
cp -a SKILL.md references ~/.codex/skills/profiling-workflow/
```

安装后，在规划或复盘 GPU profiling 工作、解释 NSYS/NCU 证据、撰写 profiling 报告时使用这个 skill。

### 设计规则

- 从理论 workload 出发，而不是从 profiler 输出出发。
- 采集 trace 前先运行无 profiler 的 baseline。
- 把编译器/运行时后端视为条件项。
- 用 NSYS 定位时间结构并验证边界。
- 只有当 GPU kernel 工作需要解释时才使用 NCU。
- 区分真实计时与 NCU replay counter。
- 项目特定脚本、路径、硬件数值和原始结果不应写入 skill。

基于新的 profiling 项目更新 skill 前，请先阅读 `references/maintenance.md`。

[Switch to English](#english)

## English

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

### Contents

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

### Install Locally

Copy the repository contents to your Codex skills directory:

```bash
mkdir -p ~/.codex/skills/profiling-workflow
cp -a SKILL.md references ~/.codex/skills/profiling-workflow/
```

After installation, use the skill when planning or reviewing GPU profiling work, interpreting NSYS/NCU evidence, or writing profiling reports.

### Design Rules

- Start from theoretical workload, not profiler output.
- Run a no-profiler baseline before collecting traces.
- Treat compiler/runtime backends as conditional.
- Use NSYS to locate time and validate boundaries.
- Use NCU only when GPU kernel work needs explanation.
- Separate real timing from NCU replay counters.
- Keep project-specific scripts, paths, hardware values, and raw results outside the skill.

See `references/maintenance.md` before updating the skill based on a new profiling project.

[切换到中文](#中文)
