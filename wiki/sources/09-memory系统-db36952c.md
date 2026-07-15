---
id: page_bbef8f825524412771e0a1e7db36952c
type: source
title: "Memory 系统"
status: draft
visibility: private
sources:
  - source_id: src_bbef8f825524412771e0a1e7db36952c
    kind: obsidian
    path: "09-Memory系统.md"
    revision: sha256:8dd360b1fd23ec2f37835595396a23eaad8644c9d4faa6e8a7624708801d473f
raw_snapshot: raw/sources/obsidian/8d/8dd360b1fd23ec2f37835595396a23eaad8644c9d4faa6e8a7624708801d473f.md
parser_id: sage.markdown
parser_version: 1.0.0
parsed_document_id: pdoc_243ad6a7350538352efd8087ecabeb5b
---

# Memory 系统

> 本页是可审核的来源投影。后续 LLM 综合必须继续保留来源 revision。

## 来源内容

---
title: 09 - Memory 系统
type: learning
project: Sage
source_branch: dev/sage-v6
status: partial
verified_at: 2026-07-10
tags: [Sage, Memory, WorkingMemory, DurableMemory, Dream]
---

# Memory 系统

> [!abstract] 本章目标
> 分清 working memory、durable memory、remember 与 dream，并理解 provenance 为什么是安全要求。

## MemoryManager 的组合

`MemoryManager` 同时持有：

- `WorkingMemory`：每次 run 从 session 证据重建。
- `DurableMemory`：按 workspace ID 保存到磁盘。
- pending dream proposal：等待批准或拒绝。

Runtime 在 run 开始时：

```text
build_working_memory(session, modes)
  -> get_context_block()
  -> Engine -> ContextManager
```

Memory block 只注入本轮 prompt，不写回 history。

## Working Memory

当前从 session history 提取：

- 最近一个用户任务摘要
- 最近错误
- 最近读写过的最多 8 个文件
- permission mode
- plan mode

它是派生状态，丢失后可以重建，不需要独立长期保存。

`recent_files` 类型包含 hash 字段，但当前 `from_session()` 生成的是空 hash；真正的文件 freshness 关联仍不完整。

## Durable Memory

workspace ID 来自 canonical path 的 SHA-256 前 16 位。存储结构：

```text
<storage_root>/memory/<workspace_id>/
├── MEMORY.md
├── daily/YYYY-MM-DD.md
├── project-conventions.md
└── decisions.md
```

事实字段包括 topic、content、source、source_ref、created_at、reviewed_at 和 status。Topic 文件当前使用 JSONL，`MEMORY.md` 是重新生成的短索引。

## Explicit Remember

`remember` 工具代表明确用户意图：

```text
用户要求记住
  -> memory tool
  -> MemoryManager.remember
  -> daily log + topic JSONL
  -> rebuild MEMORY.md
```

`source_ref` 应记录当前 run ID，方便以后知道事实从哪里产生。没有 provenance 的记忆很难判断是否可信。

## Dream Proposal

`dream` 的正确语义不是自动改记忆：

```text
读取已有 facts
  -> 生成 proposed facts
  -> 保存 pending proposal
  -> memory_proposal_ready
  -> 用户 approve / reject
```

只有 approve 才写入 durable topic。Reject 只清空 pending proposal。

当前 `propose_dream()` 基本上复制现有 facts 并标成 proposed，还没有真正的 LLM consolidation、去重或冲突处理。因此它是审批闭环原型，不是成熟的“睡眠整理系统”。

## Memory、RAG、知识图谱

| 系统 | 输入 | 输出 |
| --- | --- | --- |
| Memory | 用户确认的约定与决策 | 少量稳定事实 |
| Code RAG | 大量代码 chunk 与 query | 当前问题相关代码 |
| 知识图谱 | AST/符号/依赖关系 | 可遍历结构关系 |

V6 只做第一项。RAG 与知识图谱属于 V8，不应偷偷扩大 MemoryManager。

## 测试入口

- `tests/core/coding/test_memory.py`
- `tests/evals/test_benchmark.py::test_memory_continuity_recalls_in_second_session`
- `api/coding.py::approve_memory_proposal`
- `api/coding.py::reject_memory_proposal`

重点测试：

- workspace 隔离
- 新 session 仍能读取 explicit memory
- context budget
- source_ref
- proposal 不提前 mutation
- approve/reject

## 当前复盘点

- durable selection 当前只是裁剪 `MEMORY.md`，没有 relevance ranking。
- working recent file hash 尚未真正填充。
- dream 会重复已有 facts，缺少去重和 reviewed_at 更新。
- 前端已出现 Memory section，但 proposal review 是否形成完整可见闭环需要持续验收。

## 自测

1. 为什么 `/remember` 可以直接写，而 `/dream` 必须先 proposal？
2. workspace path hash 能解决什么隔离问题，不能解决什么安全问题？
3. 为什么 Memory 不应承担 Code RAG？

下一章：[[10-Session-Run与持久化]]
