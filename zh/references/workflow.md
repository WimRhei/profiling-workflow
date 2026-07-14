# Profiling Workflow

按下面顺序执行。每一层都应该解释为什么需要进入下一层。

## 1. 理论 workload

必需。

收集：

- 模型 stage 和依赖关系；
- input shape、batch、sequence length、action chunk、denoise/iteration count；
- 尽可能获得 GEMM/attention `M/N/K`；
- 理论 FLOP、tensor traffic、roofline lower bound；
- layer/step 之间预期的重复结构。

指标组：

| 组 | 回答什么 |
|---|---|
| Shape | 模型要求 GPU 解决什么问题规模？ |
| FLOP / MAC | 理想计算量有多少？ |
| Tensor traffic | 理想数据搬运量有多少？ |
| Roofline lower bound | 乐观延迟下限是多少？ |
| Repetition structure | kernel 行为是否应该按 layer、step、token 或 microbatch 重复？ |

目的：

- 在 profiler 噪声之前建立 workload 预期；
- 给后续每一层测量提供参照；
- 防止把 launch/tactic 数据误当作模型 shape。

输出 artifact：

| 字段 | 必要内容 |
|---|---|
| Model / config | checkpoint、模型变体、任务/input profile |
| Stage map | stage 名称、依赖关系、重复 loop/step |
| Input shape | batch、sequence/action length、image/state/noise 等 shape |
| Precision | framework/backend dtype 或量化口径 |
| Heavy ops | 预期 GEMM/attention/conv 或其他主要算子 |
| Theory numbers | 可获得的 FLOP/MAC、traffic、roofline lower bound |
| Source of truth | model code、graph export、backend layer metadata 或分析脚本 |

## 2. Framework 执行

必需。

收集：

- 与 reference output 的正确性对比；
- device/dtype/batch/shape 确认；
- 语义 stage timing；
- warmup/run 稳定性；
- 相关 power/clock/runtime 设置。

指标组：

| 组 | 回答什么 |
|---|---|
| Correctness | 模型/input/output 是否正确？ |
| Device / dtype / shape | 是否在预期 accelerator 和问题规模上运行？ |
| Baseline latency | 无 profiler 的参考耗时是多少？ |
| Warmup / variance | timing 是否稳定到可以 profiling？ |
| Semantic stage timing | 哪个模型区域可能重要？ |

目的：

- 证明模型和输入正确；
- 建立 stage 顺序和粗略瓶颈；
- 早期发现 shape 或配置漂移。

排除：

- CPU fallback；
- 错 checkpoint/config/input；
- 不稳定 warmup 或 clock 状态。

输出 artifact：

| 字段              | 必要内容                                                          |
| --------------- | ------------------------------------------------------------- |
| Correctness     | reference 对比或预期 output shape/value 检查                         |
| Runtime config  | device、dtype、batch/shape、warmup/runs                          |
| Baseline timing | latency mean/std 或代表性耗时                                       |
| Semantic stages | stage 顺序和粗略耗时，如果可获得                                           |
| Limitation      | timing 是 production path、compiled path，还是 semantic/debug path |

## 3. Compiler / Runtime Backend

条件节点。如果部署路径使用 TensorRT、ONNX Runtime、Triton、XLA、TVM、CUDA Graph 或其他 compiler/runtime 层，就执行。

收集：

- engine/model artifact、precision、binding shape、dynamic shape profile；
- layer profile 或 operator profile；
- 如适用，stage split 与 single-vs-split 对账；
- transfer policy 和 GPU-resident buffer；
- 如相关，CUDA Graph 或 runtime enqueue 行为。

指标组：

| 组 | 回答什么 |
|---|---|
| Binding / operator shapes | backend shape 是否匹配 theory/framework？ |
| Precision / quantization | backend 是否走预期数值路径？ |
| Layer/operator profile | backend 内部哪些语义区域占主导？ |
| Backend latency | backend latency 是否能和 baseline 对账？ |
| Transfer policy | 输入/中间结果是否按预期驻留？ |

目的：

- 证明被 profile 的 backend 与理论 workload 匹配；
- 验证 stage boundary 和 precision；
- 避免 profile 错 engine 或错 shape。

排除：

- engine shape mismatch；
- 意外 host/device transfer；
- stage split overhead 大到改变结论；
- layer/runtime profile 与理论不一致。

Pitfall：

- backend layer/operator 不一定对应一个 CUDA kernel。layer profile 可用于语义归因，不能当一一对应 kernel map。

输出 artifact：

| 字段 | 必要内容 |
|---|---|
| Backend artifact | engine/model/profile 路径或 runtime 配置 |
| Shape profile | binding 或 operator input shape |
| Precision | 主要 stage 的 quantization/dtype |
| Baseline reconciliation | backend 与 framework 或 production baseline 的差异 |
| Layer/operator profile | 可获得的 top operators/layers |
| Boundary | backend profile 能证明什么，不能证明什么 |

## 4. NSYS 时间结构

进入 NCU 前必需。

收集：

- measurement window 边界；
- stage duration 和 CUDA event 一致性；
- measured window 内 CUDA memcpy/memset/allocation/synchronization；
- kernel launch density 和 API enqueue time；
- GPU idle gap 和 stream overlap；
- NVTX range 覆盖情况。

指标组：

| 组 | 回答什么 |
|---|---|
| Stage duration | 哪个区域消耗 wall-clock time？ |
| CUDA API enqueue time | CPU/runtime submit 是否昂贵？ |
| Kernel duration sum | 真正 GPU kernel work 有多少？ |
| GPU idle / gap | GPU 是否在 kernel 或 stage 之间等待？ |
| Stream overlap / concurrency | 可用 stream 并行是否被利用？ |
| Memcpy / memset / allocation / sync | measurement window 是否被污染，或被 data movement/sync 主导？ |
| NVTX / range coverage | profiling boundary 是否可信？ |

目的：

- 回答时间花在哪里；
- 判断 launch/gap/transfer/sync 是否主导；
- 在采硬件 counter 前验证 profiling 边界。

排除：

- setup/warmup 污染；
- 意外 H2D/D2H/D2D copy；
- 不覆盖 GPU work 的 NVTX range；
- 当证据不支持时，排除 CPU enqueue 或 sync 是主因。

Pitfall：

- CPU-side NVTX range 可能只覆盖 enqueue，而不是 GPU completion。用 CUDA event 或 kernel timestamp 验证后，再用于 NCU filtering。

输出 artifact：

| 字段 | 必要内容 |
|---|---|
| Window | measured range、warmup/setup 是否分离 |
| Stage timing | CUDA event 或可信 wall-clock timing |
| Timeline checks | copy、memset、allocation、sync、stream 行为 |
| Launch/gap | launch density/API time、GPU idle/gap、overlap |
| Boundary verdict | window 是否可用于更深 profiling |
| Next decision | runtime/transfer/sync investigation 或 NCU |

## 5. NCU stage 和 family 分析

条件节点。只有 NSYS 表明 GPU kernel work 需要解释时才是必需。

收集：

- stage-level kernel count、可信 timing、DRAM/L2 traffic、SM throughput、memory throughput、occupancy；
- 使用 NSYS timing 做 kernel family time attribution，避免 NCU replay 时间失真；
- family-level utilization 和 traffic。

指标组：

| 组 | 回答什么 |
|---|---|
| Trusted timing | stage/family 消耗多少真实时间？ |
| Kernel count | work 是否被切成很多 kernels？ |
| DRAM read/write or bytes | 外部显存 traffic 有多少？ |
| L2 traffic / requests | cache 层 traffic 有多少？ |
| SM throughput | SM 执行资源是否忙？ |
| Memory throughput | memory system 是否忙？ |
| Occupancy | 是否有足够 resident warps 隐藏延迟？ |
| Family time share | 哪些 kernel families 值得深入？ |

目的：

- 找出重要 stage 和 kernel family；
- 避免从全局平均值下结论；
- 区分 compute、memory、low-util 和 auxiliary work。

排除：

- 当 DRAM bandwidth 低时，排除纯 memory bandwidth 上限；
- 无关的小 family；
- 把 NCU replay time 当作真实 wall-clock time。

Pitfall：

- 代表性 kernel 采样只是 smoke/debug 证据。除非覆盖率得到证明，否则不能作为 stage-level DRAM、utilization 或 top-family 结论。

输出 artifact：

| 字段 | 必要内容 |
|---|---|
| Coverage | 覆盖的 stage/window/kernel，filtering 方法 |
| Stage summary | kernel count、可信 latency、DRAM/L2、SM/memory throughput、occupancy |
| Family summary | 按可信时间排序的 top families、utilization、traffic |
| Timing source | NSYS/CUDA event timing vs NCU replay counters |
| Verdict | compute、memory、low-util、launch geometry 或 auxiliary work |

## 6. NCU per-kernel / micro-architecture

在 family analysis 找到重要 kernel 后使用。

收集：

- grid size、block size、waves/SM；
- SM throughput、TC pipe active、tensor pipe active、TMA pipe active；
- occupancy、active/eligible warps；
- issue active 和 stall breakdown；
- barrier、branch divergence、shared-memory bank conflict；
- 必要时用 source/SASS 证明 Tensor Core 或 instruction path。

指标组：

| 组 | 回答什么 |
|---|---|
| Grid / block / waves | 单次 launch 是否暴露足够 GPU-level parallelism？ |
| Registers / shared memory | block 资源是否限制 occupancy 或 tactic？ |
| SM / TC / tensor / TMA pipe | 哪些执行路径活跃，是否饱和？ |
| Active / eligible warps | scheduler 是否有 ready work？ |
| Issue active | scheduler 是否持续发射指令？ |
| Stall breakdown | 是什么阻止 ready instruction issue？ |
| Long scoreboard | 长延迟依赖是否主导？ |
| Barrier stall | 同步是否限制进度？ |
| Shared-memory conflict | shared-memory access 是否低效？ |
| Branch divergence | warp control-flow divergence 是否相关？ |

目的：

- 区分 GPU-level parallelism 和 active-SM micro-architecture 问题；
- 发现供数、依赖、barrier、branch、shared-memory 问题；
- 证明是否使用 Tensor Core，以及 Tensor Core 是否忙。

排除：

- 当 SASS/family/counter 已证明 Tensor Core 路径存在时，排除“没用 Tensor Core”；
- 当 counter 不支持时，排除 bank conflict、branch divergence 或 barrier 是根因；
- 当 waves 很高但 eligible warps 低时，不能只说 small-grid 是唯一原因。

Pitfall：

- kernel name 和 launch field 不提供完整 GEMM `M/N/K`。CUTLASS tile string 描述 tactic/tile shape，不是完整模型问题规模。

输出 artifact：

| 字段 | 必要内容 |
|---|---|
| Target kernels | family/shape/sample 覆盖率，以及为什么选它们 |
| GPU-level | grid、block、waves/SM、occupancy |
| Compute path | SM throughput、TC/tensor/TMA pipe，必要时 SASS/source |
| Scheduler | eligible warps、issue active、active warps |
| Stall/conflict | dominant stalls、barrier、branch、shared-memory conflict |
| Exclusions | 排除了哪些可疑原因 |

## 7. 可视化辅助规律提炼

辅助节点。当 kernel 行为重复、按 step 出现、漂移，或很难用聚合统计概括时使用。

使用：

- NSYS GUI 做 timeline sanity 和 stream 行为检查；
- 结合 NSYS time 和 NCU metrics 的独立 HTML/CSV timeline；
- stage/step/family/metric filter；
- SM、TC pipe、memory、waves、stall 的 per-kernel bar/timeline view。

目的：

- 判断行为是稳定、周期性、漂移、局部异常；
- 区分结构性重复和一次性异常；
- 指导最终报告应该用统计、timeline 还是代表性例子。

不要只用可视化证明硬件瓶颈。必须和 counter、理论分析配合。

输出 artifact：

| 字段 | 必要内容 |
|---|---|
| View | 使用了哪些 stage/step/family/metric filter |
| Pattern | repeated、drifting、localized 或 outlier behavior |
| Metric choice | 哪个 metric 最能暴露该 pattern |
| Reporting choice | statistic、timeline 或 representative example |

## 8. 理论-实测对齐

最终结论前必需。

对齐：

- theoretical stage workload vs measured stage timing；
- theoretical `M/N/K`、FLOP、traffic vs NCU family/kernel behavior；
- backend layer/operator profile vs CUDA kernel families；
- NSYS wall time vs NCU counters。

目的：

- 判断测量结果是由 model problem shape、runtime overhead、memory behavior 还是 micro-architecture 解释；
- 清楚说明证据边界。

如果各层对不上，先调查 shape、stage boundary、profiling window、compiler fusion、tactic change 或理论假设不支持，再下结论。

输出 artifact：

| 层 | 对齐检查 |
|---|---|
| Theory | 预期 stage、shape、FLOP/traffic |
| Framework | 正确性、实际 shape、semantic timing |
| Backend | engine/operator shape 和 latency 对账，如果使用 |
| NSYS | 真实 timing、clean window、launch/gap/stream 行为 |
| NCU | family/counter 证据、coverage、timing caveat |
| Conclusion | 支持的结论、排除项、剩余不确定性 |
