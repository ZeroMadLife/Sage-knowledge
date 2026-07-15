---
id: page_3bbed787c5626310b512dee5789b3b46
type: source
title: "Context 与 Prompt Caching"
status: draft
visibility: private
sources:
  - source_id: src_3bbed787c5626310b512dee5789b3b46
    kind: obsidian
    path: "05-Context与PromptCaching.md"
    revision: sha256:a3081504fb39361622ebc9f638f31c891979abe31b7b4d436f867b07a7791003
raw_snapshot: raw/sources/obsidian/a3/a3081504fb39361622ebc9f638f31c891979abe31b7b4d436f867b07a7791003.md
parser_id: sage.markdown
parser_version: 1.0.0
parsed_document_id: pdoc_8910ac7b0bcc4faa5fda911b815d012a
---

# Context 与 Prompt Caching

> 本页是可审核的来源投影。后续 LLM 综合必须继续保留来源 revision。

## 来源内容

---
title: 05 - Context 与 Prompt Caching
type: learning
project: Sage
source_branch: dev/sage-v6
status: verified
verified_at: 2026-07-10
tags: [Sage, ContextManager, PromptCaching, Compact]
---

# Context 与 Prompt Caching

> [!abstract] 本章目标
> 看懂一个 prompt 由哪些部分构成，哪些部分需要稳定，超预算时谁先被裁剪。

## Prompt 的五个部分

`ContextManager.build()` 最终按固定顺序组装：

```text
prefix
skill_prompt（可选）
memory（可选）
history
current_request
```

`skill_prompt` 与 `memory` 只注入本轮，不写入 session history。否则每次 resume 都会重复膨胀。

## Prefix 的三层

```text
stable：DEFAULT_SYSTEM_PROMPT + 当前 active tools
__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__
context：workspace reminders + deferred tool 名称
volatile：当天日期
```

`DEFAULT_SYSTEM_PROMPT` 包含身份、安全、做事方式、工具使用、表达风格和 XML response protocol。它是零插值常量，有利于前缀缓存。

日期只精确到天，避免同一天每一轮都破坏字节稳定性。

## 缓存键

缓存键包含：

```python
(*tools, DYNAMIC_BOUNDARY, *workspace_reminders, *deferred_tools)
```

满足以下条件才复用：

- 已有 cached system prompt
- `_system_prompt_dirty` 为 false
- tools key 完全一致

激活 deferred tool 后工具列表变化，下一轮自然重建 prefix。`system_prompt_build_count` 可以作为缓存代理指标。

## 预算分配

默认 `total_budget = 60000` 字符。当前请求、skill 和 memory 先预留，剩余空间约按 1/3 prefix、2/3 history 分配。

超预算时：

1. 先裁剪 history。
2. 再裁剪 prefix。
3. `current_request`、skill、memory 当前实现不主动裁剪。

`tail_clip()` 保留尾部，因为最近对话通常比最早对话更重要。

> [!warning] 当前边界
> 如果单个 skill 或 memory block 本身极大，reserved 可能已超过总预算；metadata 会通过 `prompt_over_budget` 暴露，但当前实现不会自动裁剪这些区块。Memory 自身通过 2000 字符预算缓解了这一点。

## Memory 的位置

Memory 位于 skill 后、history 前：

```text
system constraints
skill workflow
working + durable memory
conversation history
current request
```

这样 Memory 是有来源的辅助上下文，不会伪装成用户本轮输入。

## History Compact

`CompactManager.compact()` 按 user turn 分组：

```text
旧 turns -> compact_summary
最近 N turns -> 原样保留
```

摘要记录目标、读过的文件、修改过的文件和下一步。Compact 后调用 `ContextManager.invalidate_system_prompt()`，让下一轮重新构建一次。

这不是 LLM 摘要，目前是确定性提取，因此便宜、可测，但语义保真能力有限。

## WorkspaceContext 的 freshness

Context 模块不仅管 prompt。`WorkspaceContext` 还记录：

- `_read_fingerprints`
- `_self_authored_fingerprints`

`ToolPolicyChecker` 用它判断文件是否“刚刚读过且未被外部修改”。fingerprint 由存在性、mtime_ns、size 组成。

## 测试入口

- `tests/core/coding/test_context_compact.py`
- `tests/core/coding/test_engine.py::test_engine_ainvoke_splits_system_and_user_messages`
- `tests/core/coding/test_memory.py::test_context_injection_respects_budget`
- `tests/core/coding/test_workspace.py`

## 动手

创建一个 `ContextManager(total_budget=500)`，传入长 history、skill 和 memory，打印 metadata 的每个 section 长度。然后激活一个 deferred tool，观察 `system_prompt_build_count`。

## 自测

1. 为什么日期不能带秒？
2. 为什么 current request 不优先裁剪？
3. Compact 与 durable memory 有什么本质区别？

下一章：[[06-Tools-Skills与ToolSearch]]
