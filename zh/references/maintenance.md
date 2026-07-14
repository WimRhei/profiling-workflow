# 维护 Profiling Workflow Skill

在完成一个新的 profiling 项目后，用本文判断如何更新 skill。目标是持续优化 workflow，而不是把 skill 变成项目流水账、工具手册或 GPU 知识教材。

## 什么应该进入 Skill

加入 `SKILL.md`，当改动影响全局 workflow：

- 新的 required 或 conditional stage；
- 新的决策门槛；
- 更清晰的最小执行步骤；
- 适用于多数 NVIDIA GPU profiling 任务的新输出要求。

加入 `references/workflow.md`，当改动澄清某个阶段：

- 要收集什么；
- 为什么做这一阶段；
- 这一阶段证明什么；
- 这一阶段排除什么；
- 必看的 metric groups；
- 输出 artifact 形态。

加入 `references/evidence_matrix.md`，当新观察改变诊断路径：

- observation -> check -> next step；
- required / conditional 状态；
- 能排除某个常见错误结论的证据。

加入 `references/pitfalls.md`，当某个错误可跨项目复用：

- profiling boundary 误用；
- shape/config drift；
- timing source 错用；
- partial coverage 被当成完整证据；
- profiler 字段被误当成模型属性；
- semantic layer attribution 被夸大成 kernel attribution。

加入 `references/report_style.md`，当报告规则能提升结论质量：

- 如何陈述证据边界；
- 如何选择 statistic / timeline / representative example；
- 如何区分 timing、counter、coverage 和 uncertainty。

## 什么不应该进入 Skill

不要加入：

- 项目特定脚本或命令；
- 绝对路径；
- 某个项目的模型名、checkpoint、任务、数据集；
- 硬件特定数值，除非只是单独项目 reference 里的例子；
- 详细 CUDA、GEMM、Tensor Core 或 NCU metric 教程；
- raw experiment results；
- 一次性 debug notes；
- 过期 plan。

## 抽象规则

加入 skill 前先问：

1. 这是否适用于另一个 NVIDIA GPU profiling 项目？
2. 它是否改变诊断顺序、证据边界或输出 artifact？
3. 它是否独立于具体脚本、模型、路径、硬件 SKU 或 run date？
4. 它是否足够简洁，不是在讲完整专题？

如果答案是否定的，留在项目里。

## 如何提炼新的 profiling 经验

使用这个格式：

```text
Project observation:
  具体项目里发生了什么？

Wrong conclusion risk:
  人们可能错误推断什么？

Reusable rule:
  未来 profiling workflow 应该怎么做？

Skill destination:
  SKILL.md / workflow.md / evidence_matrix.md / pitfalls.md / report_style.md / outside skill
```

示例：

```text
Project observation:
  CPU NVTX range 只覆盖 enqueue，没有覆盖所有 GPU kernels。

Wrong conclusion risk:
  直接用 NVTX include 跑 NCU，漏掉相关 kernels。

Reusable rule:
  使用 NVTX 作为 profiling boundary 前，先用 NSYS kernel timestamp 或 CUDA event 验证。

Skill destination:
  pitfalls.md 和 workflow.md 的 NSYS boundary 阶段。
```

## 定期检查清单

每完成一个重要 profiling 项目后，检查：

- workflow 顺序是否仍然成立？
- 是否有阶段需要变成 required 或 conditional？
- 是否有新的 metric group 对诊断是必要的？
- 是否出现可复用的新 pitfall？
- 是否有新的报告边界语言避免了错误结论？
- skill 中是否混入了过于项目特定的内容，需要移出？

保持 skill 小而硬。优先写一句可复用规则，不写项目特定长段落。

