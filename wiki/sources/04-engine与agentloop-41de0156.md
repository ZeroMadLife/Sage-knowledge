---
id: page_1202dbc90d3a71108eab176d41de0156
type: source
title: "Engine 与 Agent Loop"
status: draft
visibility: private
sources:
  - source_id: src_1202dbc90d3a71108eab176d41de0156
    kind: obsidian
    path: "04-Engine与AgentLoop.md"
    revision: sha256:c68f4cfd6e0b8b730be634de66cb2f1e615751082df0599685e1cc202311a051
raw_snapshot: raw/sources/obsidian/c6/c68f4cfd6e0b8b730be634de66cb2f1e615751082df0599685e1cc202311a051.md
parser_id: sage.markdown
parser_version: 1.0.0
parsed_document_id: pdoc_80f99f7f4f1afb61de26fcc0e1408f0a
---

# Engine 与 Agent Loop

> 本页是可审核的来源投影。后续 LLM 综合必须继续保留来源 revision。

## 来源内容

---
title: 04 - Engine 与 Agent Loop
type: learning
project: Sage
source_branch: dev/sage-v6
status: verified
verified_at: 2026-07-10
tags: [Sage, Engine, AgentLoop, ApiClient]
---

# Engine 与 Agent Loop

> [!abstract] 本章目标
> 能用伪代码复述 `Engine.run_turn()`，并解释 model、parser、ToolExecutor 三者如何解耦。

## Engine 只推进一个 turn

第一入口：`core/coding/engine/engine.py::Engine.run_turn`。

```python
append user message
while within budgets:
    check stop
    build prompt
    yield model_requested
    call model
    parse raw output
    yield model_parsed

    if tool:
        detect identical repeats
        delegate ToolExecutor
        append tool result to history
        continue
    if retry:
        inject protocol correction
        continue
    if final:
        append assistant history
        yield final
        return

yield step_limit
```

Engine 不创建 session、不写 diff、不管理 WebSocket。这些都在 Runtime 外层。

## 模型客户端契约

`ApiClient` Protocol 的最小要求是：

```python
async def complete(self, prompt: str) -> str: ...
```

实际 Engine 还兼容三种路径：

- `astream(messages)`：逐块读取模型内容并产生可见的 `text_delta`。
- `complete(prompt)`：适合 ScriptedApiClient 和简单 provider。
- `ainvoke(messages)`：适合 LangChain 风格模型。

`_build_ainvoke_messages()` 按 `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` 拆成 system 与 user message。若找不到 boundary，则回退成单 user message。

## 输出协议解析

`core/coding/engine/model_output.py::parse` 把原始文本归为：

| kind | payload | Engine 动作 |
| --- | --- | --- |
| `tool` | 一个工具 payload | 执行工具并继续 |
| `tools` | 多个工具 payload | 依次执行并继续 |
| `final` | 最终文本 | 结束 loop |
| `retry` | 格式问题说明 | 给模型协议纠正信息 |

Sage 目前使用 XML 包裹的自定义协议，不是 provider 原生 function calling。因此 parser 是 harness 的关键兼容层。

## 流式文本为什么不能直接透传

模型流里可能先出现 `<final>` 标签。Engine 的 `_visible_final_delta()` 只把 final 内可见文本发给前端，避免协议标记泄漏到 UI。

非流式模型不会产生 `text_delta`，只在解析完成后产生 `final`。前端必须同时兼容两条路径。

## 两套预算

- `attempts`：调用模型的次数。
- `tool_steps`：真正执行工具的次数。

Malformed response 会消耗 attempts，但不应假装执行了工具。协议纠正最多 `MAX_PROTOCOL_RETRIES = 2`；整个 loop 的工具上限由 `max_steps` 控制。

## 重复工具调用

V6 使用“工具名 + 完整规范化 args”作为签名：

```python
sig = (tool_name, json.dumps(tool_args, sort_keys=True))
```

连续重复相同调用达到阈值后，Engine 产生 `final` 停止循环。对同一个文件写入不同内容不会被误判成重复，因为 args 不同。

## ToolExecutor 边界

Engine 通过 `_execute_tool_payload()` 委托 ToolExecutor。它只做两件额外工作：

1. 把 `ToolResultEvent` 追加为 `role=tool` 的 history。
2. 把 typed Pydantic event 转成可序列化 dict。

工具权限和审批细节不应重新写回 Engine。

## 测试入口

- `tests/core/coding/test_engine.py::test_engine_yields_tool_result_then_final`
- `test_engine_tool_search_activates_deferred_tools_for_next_prompt`
- `test_engine_ainvoke_splits_system_and_user_messages`
- `test_engine_streams_text_delta`
- `test_engine_detects_repeated_identical_write_calls`
- `tests/core/coding/test_agent_loop.py`

`ScriptedApiClient` 预置多次模型响应，可以不调用真实 API 而跑完整 loop。这比 mock Engine 内部函数更接近真实契约。

## 复盘风险

> [!warning] 自定义 XML 协议是兼容点，也是脆弱点
> Provider 输出格式稍有变化就可能进入 retry。V7 若支持更多 provider，应明确比较原生 tool calling 与当前 XML 协议的维护成本。

## 自测

1. 为什么 attempts 和 tool_steps 不能共用一个计数？
2. Engine 为什么只追加 tool result，而不自己执行工具？
3. 流式 `<final>` 标签如何避免显示给用户？

下一章：[[05-Context与PromptCaching]]
