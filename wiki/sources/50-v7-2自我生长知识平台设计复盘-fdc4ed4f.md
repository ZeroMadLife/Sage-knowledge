---
id: page_c134a03182ca7cd8bcbaae3bfdc4ed4f
type: source
title: "V7.2 自我生长知识平台设计复盘"
status: draft
visibility: private
sources:
  - source_id: src_c134a03182ca7cd8bcbaae3bfdc4ed4f
    kind: obsidian
    path: "50-V7.2自我生长知识平台设计复盘.md"
    revision: sha256:e7a1e112e2d8f8e6dba5b67772f95e54a926badbbf5ac8382634dc209ad14ac5
raw_snapshot: raw/sources/obsidian/e7/e7a1e112e2d8f8e6dba5b67772f95e54a926badbbf5ac8382634dc209ad14ac5.md
parser_id: sage.markdown
parser_version: 1.0.0
parsed_document_id: pdoc_c0b18659d1d6c753d0c2b4747b843858
---

# V7.2 自我生长知识平台设计复盘

> 本页是可审核的来源投影。后续 LLM 综合必须继续保留来源 revision。

## 来源内容

# V7.2 自我生长知识平台设计复盘

> 日期：2026-07-15
>
> Source commit：`a119778`
>
> 集成目标：`dev/sage-v7`

## 本阶段结论

Sage V7 的核心不是“云端 Coding Chat”，而是一个能持续吸收项目、笔记和外部资料，并把对话与实践结果沉淀为可追溯知识资产的个人助手。

本阶段只完成设计、架构与可交互视觉原型，没有把 P2.2-P2.4 描述成已交付功能。当前代码事实仍是 P2.1：Obsidian 单文件摄取、immutable raw snapshot、SQLite proposal/revision、approve/reject、Git Wiki 投影和 rollback proposal。

## 关键产品决策

采用 A 级自治策略，避免“所有 LLM 变更都要审核”的反人类体验：

- 低风险：自动应用，随时撤销；
- 中风险：自动写入 draft，进入每日/每周摘要，可批量撤销；
- 高风险：删除、跨主题合并、修改知识目标/Schema、HR 发布时显式确认；
- 阻断：密钥、私有会话、内部 Memory、跨租户内容进入公开知识库。

模型只参与候选分类，最终由确定性 Policy Engine 强制门限。

## 架构边界

```text
本地 / GitHub / 飞书 / Web
  -> Source Adapter
  -> Redis Streams 持久任务
  -> Parser / MinerU / Qwen3-VL
  -> Wiki Synthesizer
  -> Autonomy Policy
  -> Git Wiki + PostgreSQL 审计
  -> PostgreSQL FTS + Qdrant + Graph Tables / Louvain
  -> Sage Chat / Knowledge Workbench / HR Public Agent
```

Git Wiki 是 canonical truth；全文、向量和图谱是可重建投影。首版图谱用 PostgreSQL edge tables 与 Python Louvain，不提前引入 Neo4j；任务队列用 Redis Streams，不提前引入 Kafka。

百炼视觉解析默认选择固定版本 `qwen3-vl-flash-2026-01-22`，复杂或低置信页面升级到 Plus。凭据仅从 Keychain/服务器 Secret 注入，不进入仓库、浏览器、prompt、timeline 或 Wiki。

## 视觉门面

设计产物位于：

- `docs/assets/v7-2-knowledge-platform/sage-knowledge-workbench.html`
- `docs/assets/v7-2-knowledge-platform/sage-knowledge-workbench-concept.png`
- `docs/assets/v7-2-knowledge-platform/sage-knowledge-workbench-concept-mentalout.png`
- `docs/assets/v7-2-knowledge-platform/sage-knowledge-platform-architecture.png`

工作台由深色产品栏、知识树、主图谱画布、来源 Inspector 和底部摄取任务组成。原型支持节点详情、类型/社区着色、低风险撤销、高风险审核入口、任务展开和移动端 Drawer。

mentalout 不是标准 `/v1/images/generations` 兼容 API，但通过网页任务工作台可以使用。Key 保存在 macOS Keychain 服务 `codex-image-api-mentalout`，页面生成完成后已清空输入。仓库只记录服务名与使用边界，不保存原始 Key。

## 验证证据

- HTML parser 与必要内容检查通过；
- 内置浏览器验证节点详情联动、图谱着色、导航、Inspector、Activity Drawer 与 Escape/关闭行为；
- 控制台 error/warn 为 0；
- `1440x900`、`1024x768`、`390x844` 三视口截图通过；
- 手机横向溢出修复后 `body.scrollWidth == 390`；
- `git diff --check` 通过；
- 仓库密钥正则扫描通过；
- 架构图 PNG、SVG 与 Python 源码均可生成。

## 下一阶段

### P2.2-A：持久任务与批量来源

PostgreSQL metadata、Redis Streams job、文件夹扫描、租约、重试、dead letter、重启恢复和实时进度。

### P2.2-B：解析与自治 Wiki

Parser Registry、PDF/HTML、MinerU adapter、Qwen3-VL、Source Understanding、Workspace Synthesis 和 Autonomy Policy。

### P2.3：Hybrid RAG

Heading-aware chunk、PostgreSQL FTS、Qdrant、RRF、稳定 citation、50 条 Golden Queries 和 `retrieve_knowledge` tool。

### P2.4：Graph 与 Knowledge Workbench

Entity/Relation、evidence edge、Louvain community、Knowledge Graph API、图谱 UI、知识缺口和受控 Deep Research。

## 部署时点

推荐在 P2.2-A 完成后首次部署到 staging 服务器。这个时点持久队列、Worker、PostgreSQL migration、对象存储路径和重启恢复已经形成真实部署拓扑，适合验证 Docker Compose、GitHub Actions、域名、反向代理、日志、备份和一键回滚。单机阶段先不做负载均衡和 Kubernetes。
