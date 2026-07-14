---
name: profiling-workflow-zh
description: 用于 NVIDIA GPU profiling 的自顶向下诊断流程，覆盖模型理论、框架执行、可选编译/运行时后端、Nsight Systems、Nsight Compute，以及理论与实测对齐。适用于分析推理或训练中的 PyTorch/CUDA/TensorRT/NVIDIA GPU 性能、制定 profiling 计划、解释 NSYS/NCU 结果，或撰写边界清晰的 profiling 报告。
---

# Profiling-Workflow 中文副本

使用本 skill 诊断 NVIDIA GPU 上的推理或训练性能。目标不是立刻运行所有 profiler，而是建立一条自洽的证据链。

核心流程：

```text
1. 建立理论 workload 预期。
2. 无 profiler 跑通模型，验证 correctness、shape、dtype、device 和 baseline latency。
3. 如果使用 compiler/runtime backend，对齐 backend shape、precision、stage 和 latency。
4. 用 NSYS 定位时间花在哪里：CPU/runtime、copy/sync/gap、launch、stream，还是 GPU kernels。
5. 只有 GPU kernel work 需要解释时，才进入 NCU。
6. 在下结论前，对齐 theory、framework/backend、NSYS 和 NCU 证据。
```

## 基本纪律

- 从理论开始：模型阶段、tensor shape、batch、sequence/action length、GEMM/attention `M/N/K`、FLOP、traffic、roofline lower bound。
- 每深入一层，都检查实测行为是否与理论一致。如果不一致，先怀疑 shape、阶段边界、warmup/run、后端转换或 measurement window，不要急着下性能结论。
- compiler/runtime 后端是条件节点。如果使用 TensorRT / ONNX Runtime / Triton / XLA 等，就必须对齐；如果没有，就跳过。
- 不把项目脚本写死进 workflow。项目脚本只是本地实现方式，不能成为通用方法本身。

## 开始前

设计或执行 profiling 计划前，先读 [references/workflow.md](references/workflow.md)。

判断每一层要证明什么、排除什么时，读 [references/evidence_matrix.md](references/evidence_matrix.md)。

写 profiling 报告前，读 [references/report_style.md](references/report_style.md)。

当结论依赖 NVTX range、backend shape、NCU replay duration、代表性 kernel 采样、layer-kernel 归因或 profiler 推导 shape 时，读 [references/pitfalls.md](references/pitfalls.md)。

基于新的 profiling 项目更新本 skill 前，读 [references/maintenance.md](references/maintenance.md)。

## 最小执行清单

1. 建立理论 workload artifact：stage、shape、dtype、batch/sequence/action length、预期重算子、可获得的 FLOP/traffic。
2. 无 profiler 跑通一次，记录正确性、device、dtype、shape、warmup/run、baseline latency。
3. 如果使用 compiler/runtime 后端，验证 backend shape/profile/precision，并与 framework 或 production baseline 对账。
4. 短跑 NSYS，验证 measurement window、stage boundary、copy、sync、gap、stream 行为和 launch 密度。
5. 根据 NSYS 决策：
   - 如果 CPU enqueue、copy、sync、allocation 或 idle gap 已解释问题，停留在 NSYS/runtime 层；
   - 如果 GPU kernel work 仍需要解释，再采 NCU。
6. NCU 先做 stage/family summary，再进入 per-kernel 或 micro-architecture。至少覆盖 timing source、traffic、throughput、occupancy、launch geometry、scheduler readiness、stall reasons 和相关 compute pipes。
7. 当行为可能重复、漂移或被平均值掩盖时，使用可视化。
8. 写 alignment table：theory vs framework/backend vs NSYS vs NCU。
9. 陈述结论、已排除内容、仍未证明内容。

## 决策树

```text
从 theory + no-profiler baseline 开始。

如果 correctness、shape、dtype、device 或 baseline 不对：
  先修正运行，不进入 profiling。

如果使用 compiler/runtime backend，且它和 framework/baseline 对不上：
  先修正 backend shape、precision、stage split 或 runtime 配置。

短跑 NSYS。

如果 CPU enqueue、copy、sync、allocation 或 idle gap 已经解释耗时：
  停留在 runtime / data movement / scheduling 分析。

如果 measurement window 被污染，或 stage boundary 不可信：
  先修正 profiling boundary，不进入 NCU。

如果 GPU kernel work 主导且仍未解释：
  采 NCU stage/family 数据。

如果一两个 kernel family 主导：
  只对这些 family 进入 per-kernel 和 micro-architecture。

如果行为按 step/layer 重复，或平均值掩盖结构：
  用 timeline visualization 判断该报告统计值、时间线还是代表性例子。

最终结论前：
  对齐 theory、framework/backend、NSYS timing 和 NCU counters。
```

## 输出要求

一个好的 profiling 答案应该说明：

- 理论 workload 预期是什么；
- 每个测量层证明了什么；
- 每个测量层排除了什么；
- 当前证据不支持哪些更强结论；
- 结论属于 CPU/runtime overhead、transfer/sync/gap、GPU kernel work、GPU-level parallelism、memory behavior、micro-architecture，还是 model problem shape。
