---
id: page_2dca768c43539b3ca854367eeb36c03f
type: source
title: "Benchmark 与测试体系"
status: draft
visibility: private
sources:
  - source_id: src_2dca768c43539b3ca854367eeb36c03f
    kind: obsidian
    path: "14-Benchmark与测试体系.md"
    revision: sha256:4909a162745ed8abca8d77f20414c6d3cce07081b4e49515f883677fa102388f
raw_snapshot: raw/sources/obsidian/49/4909a162745ed8abca8d77f20414c6d3cce07081b4e49515f883677fa102388f.md
parser_id: sage.markdown
parser_version: 1.0.0
parsed_document_id: pdoc_631e87a2a64d84d5e1d5fe240a744042
---

# Benchmark 与测试体系

> 本页是可审核的来源投影。后续 LLM 综合必须继续保留来源 revision。

## 来源内容

---
title: 14 - Benchmark 与测试体系
type: learning
project: Sage
source_branch: dev/sage-v6
status: implemented-informational
verified_at: 2026-07-10
tags: [Sage, Benchmark, Testing, Eval, ScriptedApiClient]
---

# Benchmark 与测试体系

> [!abstract] 本章目标
> 分清单元测试、运行时集成测试和 benchmark；理解为什么 benchmark 必须穿过 CodingRuntime。

## 测试金字塔

### 单元测试

覆盖确定性边界：

- model output parser
- tool schema
- permission/policy
- context budget
- workspace path
- memory file store
- event reducer

### Harness 集成测试

使用 `ScriptedApiClient` 预置模型响应，运行 Engine 或 CodingRuntime：

```text
prompt -> scripted tool call -> ToolExecutor -> filesystem -> scripted final
```

不调用真实模型，因此稳定、快速、可断言完整事件序列。

### Benchmark

在隔离临时 workspace 中运行场景，检查：

- 文件最终状态
- 工具调用序列
- policy/approval
- run_finished
- workspace diff
- 跨 session memory
- 延迟和聚合指标

## 十个确定性场景

| 类别 | 场景 |
| --- | --- |
| read_explain | read-readme、read-source-file、trace-call-path |
| controlled_edit | fix-typo、add-test、add-function |
| policy_boundary | plan-mode-blocks-write、default-mode-requires-approval |
| memory_continuity | remember-test-command、remember-convention |

每个场景包含 prompt、scripted responses、初始文件、期望文件、期望工具和 policy 断言。

## 为什么要跑 CodingRuntime

如果 benchmark 直接调用 Engine，会绕过：

- active-run lease
- run trace
- session 分区
- workspace diff artifact
- memory manager
- run_finished
- finally cleanup

当前 `evals/coding/runner.py` 通过 `_build_runtime()` 构造真实 CodingRuntime，并消费 `runtime.run_turn()`。这才是在评估 harness，而不仅是 parser + tools smoke test。

## Approval 自动响应

Policy 场景可能让 Runtime 停在 approval wait。Runner 的 `_run_turn_with_auto_approval()` 在收集事件的同时检查 pending approval，并按场景预设响应。

它模拟真实异步闭环，而不是把 permission mode 偷改成 auto 让测试绕过去。

## Memory 连续性

Memory 场景必须：

1. 在 workspace A 的 session 1 记住事实。
2. 新建 session 2。
3. 确认 context 能召回。
4. 确认其他 workspace 不出现该事实。

只在同一个 session 内检查 history 不算 durable memory benchmark。

## 当前指标

- `task_completion_rate`
- `tool_call_success_rate`
- `policy_compliance_rate`
- `p95_turn_latency_ms`

设计书里还提出 first-pass test success、diff attribution、memory recall accuracy 等更细指标；当前 metrics.py 尚未全部实现。

## 报告

Runner 生成：

- JSON
- Markdown
- HTML

Benchmark 被明确标记为 informational，不作为 CI 硬 gate。原因是当前 scripted 场景能验证契约，但不能代表真实 provider 的智能水平。

## 测试入口

- `tests/evals/test_benchmark.py`
- `evals/coding/scenarios.py`
- `evals/coding/runner.py`
- `evals/coding/assertions.py`
- `evals/coding/metrics.py`
- `evals/coding/run.sh`

运行：

```bash
python -m evals.coding.runner
```

## 如何判断 Benchmark 是否在“自我欺骗”

检查四件事：

1. Runner 是否走真实 Runtime，而不是直接执行期望 patch。
2. Assertions 是否检查文件和 trace，而不只是“有 final”。
3. Memory 是否跨新 session。
4. 场景结果是否把 scripted model 的确定性与真实模型能力分开标记。

## 当前边界

十个 scripted 场景主要验证 harness contract，不能证明真实 coding success rate。后续应增加独立标识的 real-provider eval，但不应让不稳定外部 API 阻塞基础 CI。

## 自测

1. 为什么 10/10 scripted PASS 不等于真实用户任务成功率 100%？
2. 直接驱动 Engine 会漏测哪些 V6 能力？
3. 哪些指标当前只在设计书里，尚未落到 metrics.py？

下一章：[[15-V6源码复盘与技术债]]
