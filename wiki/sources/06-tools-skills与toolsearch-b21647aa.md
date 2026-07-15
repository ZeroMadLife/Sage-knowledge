---
id: page_b49fe73497f4548edde46af0b21647aa
type: source
title: "Tools、Skills 与 Tool Search"
status: draft
visibility: private
sources:
  - source_id: src_b49fe73497f4548edde46af0b21647aa
    kind: obsidian
    path: "06-Tools-Skills与ToolSearch.md"
    revision: sha256:c861c3e2484c3604cc2701458931d28679f85cb58752bdbab173b66887542a16
raw_snapshot: raw/sources/obsidian/c8/c861c3e2484c3604cc2701458931d28679f85cb58752bdbab173b66887542a16.md
parser_id: sage.markdown
parser_version: 1.0.0
parsed_document_id: pdoc_b0a2f70a5410674a9c8c62e98686a101
---

# Tools、Skills 与 Tool Search

> 本页是可审核的来源投影。后续 LLM 综合必须继续保留来源 revision。

## 来源内容

---
title: 06 - Tools、Skills 与 Tool Search
type: learning
project: Sage
source_branch: dev/sage-v6
status: verified
verified_at: 2026-07-10
tags: [Sage, Tools, Skills, ToolSearch]
---

# Tools、Skills 与 Tool Search

> [!abstract] 本章目标
> 分清 Tool 与 Skill：一个执行动作，一个改变模型本轮的工作方式。

## Tool 是可执行能力

`@register_tool(...)` 把函数注册成 `ToolDefinition`：

```text
name + description + schema
schema_model
risky / requires_approval
category
timeout
deferred
handler
```

`build_tool_registry()` 再把 definition 绑定到当前 `WorkspaceContext` 和 `ToolContext`，变成 `RegisteredTool`。

工具大致分为：

- 文件：`list_files`、`read_file`、`search`、`write_file`、`patch_file`
- Shell：`run_shell`
- 任务：`todo_add/update/list`
- Plan：`enter_plan_mode`、`exit_plan_mode`
- Agent：`agent`、`send_message`、`task_stop`
- Memory：`remember`、`dream`
- 领域能力：旅游规划、天气、地图等

## 三层参数保护

工具参数不是拿到 dict 就执行：

1. Pydantic schema 验证类型和必填字段。
2. `validate_tool()` 验证 workspace 相关条件。
3. ToolExecutor 再跑 permission 与 policy。

例如 `patch_file` 要求目标存在，而且 `old_text` 必须恰好出现一次。这样模型不能用含糊 patch 猜着修改。

## Deferred Tool

所有工具 schema 全塞进 prompt 会增加 token，并干扰模型选择。Deferred 工具默认不出现在 active tool description 中。

```text
模型发现当前工具不够
  -> 调 tool_search(query)
  -> 匹配 name / description / category
  -> 将工具名加入 activated_tools
  -> 下一轮 prompt 出现完整 schema
```

`activated_tools` 保存进 session，因此 resume 后仍然有效。

## Skill 是 Prompt Workflow

Skill 来源优先级：

```text
builtin
  -> ~/.sage/skills
  -> <workspace>/skills
  -> <workspace>/.coding/skills
```

后加载的同名 Skill 覆盖前面的版本。`SKILL.md` frontmatter 描述名称、说明、允许工具、参数提示等，正文是需要注入模型的工作流。

调用：

```text
/review core/coding/runtime.py
```

`SkillRegistry.resolve()` 得到 skill 和 arguments，`Skill.render()` 替换 `$ARGUMENTS` 等变量。渲染后的内容作为 `skill_prompt` 只注入当前 turn。

当前 bundled skills 包括 review、test、commit、planmode、remember、dream 和旅游相关工作流。

## Tool 与 Skill 的区别

| Tool | Skill |
| --- | --- |
| Python 可执行函数 | Markdown prompt workflow |
| 有 schema 和 ToolResult | 有 frontmatter 和正文 |
| 必须经过 ToolExecutor | 由 API 展开后注入 Context |
| 直接改变文件或状态 | 指导模型怎样组织多个动作 |
| 可能 deferred | 可通过 slash command 调用 |

Skill 不能绕过权限。即使 skill 文本要求删除文件，真正的删除动作仍必须走 ToolExecutor。

## 当前边界

- `allowed_tools` 已被解析并展示为 metadata，但当前主执行链没有据此建立严格的临时工具白名单。
- Tool registry 通过模块 import 产生全局 definition；每个 runtime 绑定的是独立 workspace runner。
- `RegisteredTool.execute()` 在线程池内同步执行并有 timeout，但 timeout 并不自动杀死已经在线程中运行的底层操作。

## 测试入口

- `tests/core/coding/test_tools.py`
- `tests/core/coding/test_skills.py`
- `tests/core/coding/test_engine.py::test_engine_tool_search_activates_deferred_tools_for_next_prompt`
- `tests/core/coding/test_todo_plan_worker_runtime.py::test_runtime_persists_activated_deferred_tools`

## 自测

1. 为什么 `tool_search` 自己不能 deferred？
2. Skill 为什么不能直接获得写文件权限？
3. `ToolDefinition` 与 `RegisteredTool` 的区别是什么？

下一章：[[07-ToolExecutor与安全治理]]
