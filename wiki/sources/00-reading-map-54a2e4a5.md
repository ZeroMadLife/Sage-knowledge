---
id: page_73c4a9a137d688d614b774bd54a2e4a5
type: source
title: "Sage Learning 阅读地图"
status: draft
visibility: private
sources:
  - source_id: src_73c4a9a137d688d614b774bd54a2e4a5
    kind: obsidian
    path: "00-reading-map.md"
    revision: sha256:1451a2cb585e5fbdf55e62d1872f1d90c2443d9e4713d32ac10430da47207c98
raw_snapshot: raw/sources/obsidian/14/1451a2cb585e5fbdf55e62d1872f1d90c2443d9e4713d32ac10430da47207c98.md
parser_id: sage.markdown
parser_version: 1.0.0
parsed_document_id: pdoc_0e036f92e4cedaacdf55b28e1034de6e
---

# Sage Learning 阅读地图

> 本页是可审核的来源投影。后续 LLM 综合必须继续保留来源 revision。

## 来源内容

---
title: Sage Learning 阅读地图
type: learning-index
project: Sage
source_project: /Users/zeromadlife/Desktop/tour-agent
source_branch: dev/sage-v6
verified_at: 2026-07-12
tags: [Sage, learning, coding-agent, harness, 源码阅读]
---

# Sage Learning 阅读地图

这套笔记不是 Sage 的宣传文档，也不是把每个 Python 文件翻译成中文。它要解决的问题是：**我们虽然通过 Vibe Coding 做出了 Sage，但如何真正看懂它、验证它，并能自己修改它。**

阅读时始终把 Sage 看成一个 coding agent harness：

```text
模型能力
  + 工具执行
  + 上下文管理
  + 权限与审批
  + 运行状态
  + 可追溯证据
  + Web 交互
= 一个可以连续完成编码任务的系统
```

## 源码事实口径

- 项目：`/Users/zeromadlife/Desktop/tour-agent`
- 分支：`dev/sage-v6`
- 主要后端：`core/coding/`、`api/coding.py`
- 主要前端：`frontend/src/views/CodingView.vue`、`frontend/src/stores/coding*.ts`、`frontend/src/components/coding/`
- 验证入口：`tests/core/coding/`、`tests/api/test_coding_routes.py`、`tests/evals/test_benchmark.py`
- 产品设计：`docs/superpowers/specs/2026-07-10-sage-v6-harness-design.md`

文档里的状态标记含义：

| 标记 | 含义 |
| --- | --- |
| **已实现** | 当前源码和测试里都能找到对应行为 |
| **部分实现** | 主路径存在，但能力、交互或正确性尚未完整 |
| **规划中** | 只出现在设计书或 V7/V8 路线里 |
| **复盘风险** | 源码存在，但仍应重点审查或补强 |

设计书描述“应该是什么”，源码描述“现在是什么”。二者冲突时，本手册以源码和测试为事实，并把差异写进“当前边界”。

## 推荐阅读顺序

### 第一遍：建立全局地图

1. [[01-Sage总体架构]]
2. [[02-一次请求的完整生命周期]]
3. [[03-CodingRuntime运行时组装]]
4. [[04-Engine与AgentLoop]]

这一遍不要追逐每个函数，只回答三个问题：请求从哪里进来、谁推进任务、证据最终存在哪里。

### 第二遍：理解 Harness 为什么可靠

5. [[05-Context与PromptCaching]]
6. [[06-Tools-Skills与ToolSearch]]
7. [[07-ToolExecutor与安全治理]]
8. [[08-Workspace-Git与Diff]]
9. [[09-Memory系统]]
10. [[10-Session-Run与持久化]]

这一遍重点看边界：模型不能直接写文件，工具不能绕过权限，Memory 不能无来源地注入，Run 结束必须有终态证据。

### 第三遍：理解网页产品闭环

11. [[11-多智能体系统]]
12. [[12-API-WebSocket与事件协议]]
13. [[13-前端CodingWorkbench]]
14. [[14-Benchmark与测试体系]]

这一遍要能从 `CodingComposer.send()` 一路追到 `CodingRuntime.run_turn()`，再回到 `applyCodingEvent()`。

### 第四遍：开始真正动代码

15. [[15-V6源码复盘与技术债]]
16. [[16-源码组件速查表]]
17. [[17-动手实验]]

不要把“看完笔记”等同于“看懂源码”。至少完成 3 个动手实验，并能不看文档画出一次 run 的事件时序。

### 第五遍：用竞品反推下一版设计

18. [[18-上下文压缩对标研究]]
19. [[19-长短记忆与Dream对标研究]]
20. [[20-Hermes-Studio前端对标研究]]
21. [[21-Sage-V6.6-V6.8上下文记忆Dream设计]]
22. [[22-Sol子Agent开发顺序与验收]]

这一遍不再只是解释当前源码，而是把 Claude Mini、Hermes Agent、OpenClaw 和 Hermes Studio 的成熟机制拆开比较。阅读时要区分“值得采用的机制”“明确拒绝的风险”和“Sage 当前还没有实现的契约”。

### 第六遍：把开发结果安全收口

23. [[27-V6.9持久时间线与运行重连复盘]]
24. [[28-V6.9中文工作台与会话导航复盘]]
25. [[29-V6.9交付收口与下一阶段预告]]
26. [[30-V6.9阶段产出与Git收口记录]]
27. [[31-Sage个人AI学习助手重定位与路线校准]]
28. [[32-Sage-Learning-MVP-PlanMode交接]]

这一遍理解两件事：持久事件如何让界面恢复，以及为什么通过单个 worktree 测试不等于可以删除它。每次 Git 合并都要依据第 30 章留下证据。

### 第七遍：从最终集成点重建 V6.9 前端全景

29. [[41-V6.9前端完整交付地图]]

这一遍不再按单个功能分支阅读，而是从 `dev/sage-v6@2016852` 把工作台界面、timeline projection、会话隔离、恢复机制和 Provider 控制面串成一张完整地图。适合作为进入 V7 Learning Practice Engine 之前的前端基线。

## 一张图记住主线

![[sage-v6-architecture.png]]

最重要的主链路：

```text
CodingView / CodingComposer
  -> Pinia useCodingStore
  -> CodingStream(WebSocket)
  -> api/coding.py::coding_stream
  -> CodingRuntime.run_turn
  -> Engine.run_turn
  -> ToolExecutor.execute
  -> RegisteredTool.execute
  -> typed RunEvent
  -> RunStore trace + WebSocket
  -> applyCodingEvent
  -> 聊天、审批、Diff、Run History 更新
```

## 读源码的方法

每章都按同一个动作读：

1. 先打开章节列出的“第一入口”。
2. 只跟一条调用链，不同时展开所有 import。
3. 找到状态落点：内存对象、session JSON、trace JSONL 或 diff artifact。
4. 打开对应测试，观察测试怎样构造输入、怎样断言事件。
5. 完成本章的一个动手实验。
6. 用自己的话回答章末自测题。

## 不要混淆的三套系统

| 系统 | 回答的问题 | 当前阶段 |
| --- | --- | --- |
| Durable Memory | 用户明确要求 Sage 长期记住什么 | V6 已实现基础文件版 |
| Code RAG | 当前问题应该检索哪些代码片段 | V8 规划 |
| AST 知识图谱 | 类、函数、文件之间如何结构化连接 | V8 规划 |

Memory 不是 RAG，RAG 也不是知识图谱。后两者不能因为“听起来像记忆”就塞进 `core/coding/memory/`。

## 相关资料

- [[15-V6源码复盘与技术债|当前实现复盘]]
- [[16-源码组件速查表|组件到源码的速查表]]
- [[17-动手实验|动手练习]]
- [[18-上下文压缩对标研究|Claude Mini、Hermes、OpenClaw 压缩对标]]
- [[19-长短记忆与Dream对标研究|Memory 与 Dream 治理对标]]
- [[20-Hermes-Studio前端对标研究|Hermes Studio 前端对标与许可证边界]]
- [[21-Sage-V6.6-V6.8上下文记忆Dream设计|方案 B 正式架构与门限]]
- [[22-Sol子Agent开发顺序与验收|多 Agent 开发波次与文件所有权]]
- [[27-V6.9持久时间线与运行重连复盘|持久时间线与运行重连]]
- [[28-V6.9中文工作台与会话导航复盘|中文工作台与会话导航]]
- [[29-V6.9交付收口与下一阶段预告|V6.9 交付边界]]
- [[30-V6.9阶段产出与Git收口记录|阶段产出与 Git 收口]]
- [[31-Sage个人AI学习助手重定位与路线校准|个人 AI 学习助手定位与 V7/V8 路线校准]]
- [[32-Sage-Learning-MVP-PlanMode交接|后续会话可直接读取的 Plan Mode 交接]]
- [[41-V6.9前端完整交付地图|V6.9 前端完整交付地图]]
- 架构图可编辑源：`_assets/sage-v6-architecture.svg`
