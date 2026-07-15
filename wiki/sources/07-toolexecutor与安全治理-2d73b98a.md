---
id: page_a9f96d8e509d6f93b461ba1d2d73b98a
type: source
title: "ToolExecutor 与安全治理"
status: draft
visibility: private
sources:
  - source_id: src_a9f96d8e509d6f93b461ba1d2d73b98a
    kind: obsidian
    path: "07-ToolExecutor与安全治理.md"
    revision: sha256:000446246ad67ace4f83c3719d9a3edd31fa4263c0017186b791ff62bf164886
raw_snapshot: raw/sources/obsidian/00/000446246ad67ace4f83c3719d9a3edd31fa4263c0017186b791ff62bf164886.md
parser_id: sage.markdown
parser_version: 1.0.0
parsed_document_id: pdoc_386633516c369c2331e02662ec5be048
---

# ToolExecutor 与安全治理

> 本页是可审核的来源投影。后续 LLM 综合必须继续保留来源 revision。

## 来源内容

---
title: 07 - ToolExecutor 与安全治理
type: learning
project: Sage
source_branch: dev/sage-v6
status: verified
verified_at: 2026-07-10
tags: [Sage, ToolExecutor, Permission, Policy, Approval]
---

# ToolExecutor 与安全治理

> [!abstract] 本章目标
> 看懂“模型想执行”到“工具真的执行”之间的每一道门。

## 固定执行管线

`ToolExecutor.execute()`：

```text
stop check
  -> normalize payload
  -> find tool
  -> validate args
  -> PermissionChecker
  -> ToolPolicyChecker
  -> ApprovalManager（如需要）
  -> tool_call
  -> RegisteredTool.execute
  -> stop check
  -> tool_result
```

任何失败都被转成 typed `ToolResultEvent`，而不是让普通参数错误炸掉整个 WebSocket。

## Permission：动作能不能做

四种模式：

| 模式 | 文件编辑 | 普通 Shell | 危险 Shell |
| --- | --- | --- | --- |
| `default` | 请求审批 | 请求审批 | 请求审批 |
| `accept_edits` | 自动允许 | 请求审批 | 请求审批 |
| `auto` | 自动允许 | 自动允许 | 仍请求审批 |
| `plan` | 禁止 | 禁止 | 禁止 |

只读工具优先允许。Worker 还可配置 `write_scope`，写入路径必须落在指定子树。

`approval_policy=never` 是硬拒绝，不能被 mode 绕过。

## Policy：做法是否符合编码规范

`ToolPolicyChecker` 当前处理：

- `patch_file` 前必须 fresh read。
- 覆盖已有文件的 `write_file` 前必须 fresh read。
- 普通搜索/读取不应通过 shell 的 cat、grep、rg、find、ls 完成。

Permission 回答“有没有权做”，Policy 回答“当前做法是否合理”。两者不要合并成一个巨大 if/else。

## Approval：等待用户决定

`ApprovalManager.submit()` 创建 `ApprovalEntry`，其中有 `threading.Event`。ToolExecutor：

1. 产生 `approval_required`。
2. 用 `asyncio.to_thread(entry.event.wait, 1.0)` 等待。
3. 前端调用 REST approval API。
4. `resolve()` 写 choice 并 set event。
5. ToolExecutor 产生 `approval_granted` 或错误结果。

等待最长 300 秒，同时每秒检查 Stop。

审批选择：

- `once`：只批准当前动作。
- `session` / `always`：当前实现都记入 session 级 pattern allow set。
- `deny`：返回错误 tool result。

“always”当前没有跨 session 的永久语义，这是命名与实现需要注意的边界。

## 危险 Shell

`check_dangerous_command()` 用模式识别：

- 递归删除
- `git reset --hard`
- force push
- `sudo`
- `curl | sh`
- 写 `/etc`、`~/.ssh`
- 强杀进程等

即使 permission mode 是 `auto`，危险 Shell 仍转成 `approval_required`。

## Plan Mode 不是提示词

Plan mode 通过 `PermissionChecker` 强制只读。`exit_plan_mode` 只创建 `PlanReviewEntry`，不直接退出；用户通过 REST approval 后 Runtime 才切回 default。

这比在 system prompt 里写“请不要改文件”可靠，因为策略位于执行边界。

## 测试入口

- `tests/core/coding/test_tool_executor.py`
- `tests/core/coding/test_permissions.py`
- `tests/core/coding/test_approval.py`
- `tests/core/coding/test_plan_review.py`
- `tests/core/coding/test_todo_plan_worker_runtime.py::test_run_turn_emits_plan_ready_for_review_on_plan_exit`

## 自测

1. 为什么 Policy denial 应返回 tool result，而不是抛出异常？
2. `auto` 为什么仍不能自动执行危险 Shell？
3. Plan mode 为什么必须在 PermissionChecker 再检查一次？

下一章：[[08-Workspace-Git与Diff]]
