---
id: page_e38fe58fff5f697fafc3510675740fd7
type: source
title: "CodingRuntime 运行时组装"
status: draft
visibility: private
sources:
  - source_id: src_e38fe58fff5f697fafc3510675740fd7
    kind: obsidian
    path: "03-CodingRuntime运行时组装.md"
    revision: sha256:8d24e1bd796c9395b4bf169b49e17c86e4f6e0c7296be0f33b63efa564d707b0
raw_snapshot: raw/sources/obsidian/8d/8d24e1bd796c9395b4bf169b49e17c86e4f6e0c7296be0f33b63efa564d707b0.md
parser_id: sage.markdown
parser_version: 1.0.0
parsed_document_id: pdoc_77f8fdd2335d3afa807e37c095d74631
---

# CodingRuntime 运行时组装

> 本页是可审核的来源投影。后续 LLM 综合必须继续保留来源 revision。

## 来源内容

---
title: 03 - CodingRuntime 运行时组装
type: learning
project: Sage
source_branch: dev/sage-v6
status: verified
verified_at: 2026-07-10
tags: [Sage, CodingRuntime, lifecycle, composition-root]
---

# CodingRuntime 运行时组装

> [!abstract] 本章目标
> 看懂 `CodingRuntime` 为什么是 composition root，以及它与 Engine 的职责边界。

## 第一入口

`core/coding/runtime.py::CodingRuntime.__init__`

它一次性组装完整 session 所需对象：

```text
WorkspaceContext
CodingSessionStore
RunStore
WorkspaceDiffTracker
SessionEventBus
TodoLedger
PlanModeManager / PlanReviewManager
WorkerManager
ContextManager
ApprovalManager
PermissionChecker / ToolPolicyChecker
ToolContext + tool registry
SkillRegistry
MemoryManager
```

这就是 composition root：具体实现只在顶层组装，下面的 Engine 和 ToolExecutor 通过构造参数拿依赖。

## Session 状态与运行对象

需要区分两类东西。

### 可持久化状态

```python
session = {
    "id": ...,
    "workspace_root": ...,
    "history": [],
    "runtime_mode": {"mode": "default"},
    "todos": ...,
    "activated_tools": [],
    "permission_mode": ...
}
```

这些内容可以保存并通过 `CodingRuntime.resume()` 恢复。

### 进程内运行对象

`ApprovalManager`、`PlanReviewManager`、model client、active run lease 等包含线程事件、回调或活动连接，不能直接序列化。Resume 时会根据 session 状态重新创建。

## ToolContext 是反向连接

`ToolContext` 持有 `runtime`、`todo_ledger` 和 `worker_manager`。因此工具可以通过受控接口触发：

- 进入/退出 plan mode
- 新建或更新 todo
- 创建 worker
- 写入 memory

这是一种依赖反转：工具不 import 全局单例，而是执行时拿到当前 session 的 context。

## run lease

`active_run_id` 是同 session 的互斥租约：

```text
None -> 接受新请求
run_xxx -> 拒绝第二个并发请求
finally -> 恢复 None
```

重点不是“加一个变量”，而是清理位置。Async generator 可能：

- 正常消费完成
- Engine 抛异常
- WebSocket 断开
- 调用方执行 `aclose()`

因此释放必须位于不能 yield 的外层 `finally` 中。

对应测试：

- `test_run_turn_rejects_concurrent_run`
- `test_run_turn_cleanup_on_exception`
- `test_run_turn_releases_lease_on_aclose`

## run_id 绑定的 Stop

`request_stop(run_id)` 接收可选 run ID：

```text
没有 run_id       -> 为兼容旧客户端，停止当前 run
run_id 匹配       -> 设置 stop_requested
run_id 不匹配     -> 拒绝，避免延迟请求误停下一轮
```

前端应尽量携带当前 run ID。兼容路径不是长期最严格的契约。

## Runtime 与 Engine 的边界

| CodingRuntime | Engine |
| --- | --- |
| session/run 生命周期 | model/tool 循环 |
| 组装依赖 | 使用已注入依赖 |
| snapshot 与 diff artifact | 解析模型输出 |
| trace 和 session event bus | 产生 loop 级事件 |
| plan review 状态变化再注入 | 调用 ToolExecutor |
| run_finished / turn_finished | final / retry / step_limit |
| finally 清理 | 不管理 WebSocket |

一个判断方法：如果行为必须发生在模型 loop 结束之后，它通常属于 Runtime；如果行为决定下一轮该调模型还是跑工具，它通常属于 Engine。

## 事件持久化策略

Runtime 对 Engine 事件执行三件事：

```python
if event["type"] != "text_delta":
    run_store.append_trace(run_id, event)
session_event_bus.emit(event["type"], event)
yield event
```

`text_delta` 不进 trace 是为了避免 token 级事件造成 trace 膨胀。最终完整文本会由 `final` 保存。

## mode 与 plan review 的再注入

进入 plan mode 的动作发生在工具里，Engine 只看到普通 `tool_result`。Runtime 比较执行前后的 `runtime_mode`，发现变化后补发 `runtime_mode_changed`。

同理，`exit_plan_mode` 只是提交 review；Runtime 观察 `PlanReviewManager.pending`，补发 `plan_ready_for_review`。真正退出要等 REST approval。

## Resume

`CodingRuntime.resume()`：

1. 从 session store 加载 JSON。
2. 读取已保存的 `workspace_root`。
3. 重建 runtime 对象图。
4. 恢复 mode、todos、activated tools、permission mode。
5. 不应无故修改旧 session 的 `updated_at`。

跟读：`tests/core/coding/test_todo_plan_worker_runtime.py::test_runtime_resumes_persisted_session_state`。

## 当前复盘点

> [!warning] 大类风险
> `runtime.py` 已超过 500 行，同时承担对象组装、工作区 REST helper、mode 操作、run 生命周期和 session 同步。当前边界仍可理解，但 V7 引入用户 workspace/sandbox 后，应考虑把 workspace service 和 run coordinator 拆开。

## 动手

1. 画出 `CodingRuntime.__init__` 的对象依赖图。
2. 在测试里手动 `aclose()` run generator，观察 `active_run_id`。
3. 创建 session、激活 deferred tool、resume，确认它仍被激活。

## 自测

1. 为什么 ApprovalManager 不直接存进 session JSON？
2. ToolContext 解决了什么依赖问题？
3. 哪些事件只能由 Runtime 产生？

下一章：[[04-Engine与AgentLoop]]
