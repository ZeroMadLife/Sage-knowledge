---
id: page_4bd828bfa2118e3c1a5ec9da98583340
type: source
title: "Sage Learning MVP Plan Mode 交接"
status: draft
visibility: private
sources:
  - source_id: src_4bd828bfa2118e3c1a5ec9da98583340
    kind: obsidian
    path: "32-Sage-Learning-MVP-PlanMode交接.md"
    revision: sha256:bfb75775226ea8db78f41ce117e5cb87fc52e32b2e8adfcd87fdcb1736cfb20e
raw_snapshot: raw/sources/obsidian/bf/bfb75775226ea8db78f41ce117e5cb87fc52e32b2e8adfcd87fdcb1736cfb20e.md
parser_id: sage.markdown
parser_version: 1.0.0
parsed_document_id: pdoc_203e60a6ca11219c475603a20b694976
---

# Sage Learning MVP Plan Mode 交接

> 本页是可审核的来源投影。后续 LLM 综合必须继续保留来源 revision。

## 来源内容

---
title: Sage Learning MVP Plan Mode 交接
type: agent-handoff
project: Sage
source_project: /Users/zeromadlife/Desktop/tour-agent
source_branch: dev/sage-v6
source_commit: 912a3f7407c0ec22b30f9c5ac2a3959f8f3b09f1
status: ready-for-plan-mode
verified_at: 2026-07-13
tags: [sage, learning-mvp, plan-mode, handoff]
---

# Sage Learning MVP Plan Mode 交接

## 给后续会话的直接指令

你正在为 `/Users/zeromadlife/Desktop/tour-agent` 中的 Sage 制定下一阶段产品与技术计划。本轮处于 **Plan Mode**，只做调研、边界分析、领域建模和分阶段计划，不修改产品代码。

开始前依次阅读：

1. [[31-Sage个人AI学习助手重定位与路线校准]]；
2. [[29-V6.9交付收口与下一阶段预告]]；
3. [[26-学习进度与问题清单]]；
4. `/Users/zeromadlife/Desktop/tour-agent/docs/superpowers/specs/2026-07-11-sage-v6-context-memory-evolution-design.md`；
5. `/Users/zeromadlife/Desktop/tour-agent/docs/superpowers/specs/2026-07-10-sage-v6-harness-design.md`；
6. 当前 `dev/sage-v6` 源码和测试，尤其是 `core/coding/`、`api/coding.py`、`frontend/src/stores/coding*.ts` 和 `evals/coding/`。

## 已确认的产品决策

- Sage 的主产品是 **Personal AI Learning Companion**，不是通用 Coding Agent；
- 首要用户是程序员和技术学习者；
- 长期知识源覆盖代码仓库、Obsidian、Markdown、课程、论文和面试材料；
- Coding Runtime 继续存在，但定位为 **Practice Engine**；
- V6.9 继续完成可信执行、恢复、Context、Memory、Skills 权限和只读 Agent 底座；
- V7 优先 Learning MVP，本地单用户学习闭环先于多租户 SaaS 基础设施；
- V8 建设 Personal Knowledge Intelligence：个人知识 RAG、Code RAG、AST/概念图谱、Local Companion 与增量索引；
- 任何 Learning State 更新都必须绑定可验证 evidence，不能根据聊天次数或模型主观判断自动修改；
- 当前 benchmark 的 scripted 指标只证明 harness 回归，不代表真实模型或学习效果。

## 需要在 Plan Mode 中解决的问题

### 1. 第一条端到端主流程

围绕以下候选流程收敛 MVP：

```text
选择代码仓库 + Obsidian 范围
  -> 建立知识源索引
  -> 诊断一个学习目标
  -> 生成阶段学习计划
  -> 阅读 / 讲解 / 代码实验
  -> 收集 LearningEvidence
  -> Reviewer 评估
  -> 用户确认 LearningState proposal
  -> 安排下一步或复习
```

必须明确入口、终态、失败恢复和非目标。

### 2. 最小领域模型

至少定义并说明关系、生命周期和 canonical storage：

- `LearningWorkspace`
- `KnowledgeSource`
- `LearningGoal`
- `LearningPlan`
- `LearningActivity`
- `LearningEvidence`
- `LearningState`
- `LearningStateProposal`

优先使用现有 transcript、run、memory proposal 和 revision contract；不能为了复用而混淆语义。

### 3. V7 里程碑

将 V7 拆成 3 至 4 个可独立验收的里程碑。每个里程碑必须给出：

- 用户可见价值；
- 数据模型与 API 边界；
- 复用的现有模块；
- 不包含的后续能力；
- 自动化测试与手动验收；
- 量化指标；
- 进入下一个里程碑的门槛。

### 4. V8 检索与知识图谱

给出可演进的数据流，而不是直接堆技术组件：

```text
source ingestion
  -> normalization / chunk / symbol extraction
  -> sparse / dense / symbol indexes
  -> RRF / rerank
  -> provenance-preserving context
  -> answer / learning activity
  -> retrieval evaluation
```

分别说明 Personal Knowledge RAG、Code RAG、AST Graph 和 Concept Graph 的职责。每个外部组件必须给出何时才需要引入的规模或能力门槛。

### 5. 评测与安全

计划至少覆盖：

- Retrieval：`Recall@K`、`MRR`、`NDCG@K`、citation correctness；
- Harness：task/tool/policy/recovery/latency/evidence completeness；
- Learning：pre/post gain、delayed recall、independent application、calibration；
- 数据源授权、路径范围、prompt injection、秘密扫描、索引删除；
- code experiment sandbox、网络策略、资源配额和审计；
- 学习状态 proposal 的人工确认、回滚和 provenance。

## 需要显式重新排序的旧 V7 任务

对以下任务逐项给出“保留在 V7 / 推迟 / 删除”的结论与原因：

- auth；
- tenant isolation；
- cloud workspace；
- GitHub import；
- sandbox；
- quota；
- terminal；
- persistent task queue；
- writable Frontend / Backend Agent；
- CI/CD 与公网部署。

默认倾向：保留本地 sandbox 和受控 terminal；GitHub import 可作为数据源接入候选；多租户、计费、云调度后置。

## Plan Mode 约束

- 不修改产品代码，不创建 migration，不安装依赖；
- 不把本章规划写成当前实现；
- 先审源码和测试，再判断可复用性；
- 只提出能解释用户价值和验收方法的模块；
- 第一版不默认同时采用 Elasticsearch、Milvus、Neo4j、Kafka；
- Memory、RAG、Graph、Learning State 必须保持独立边界；
- 文档和计划使用中文，代码标识符与 API 字段保留英文；
- 在用户确认主流程、领域模型和里程碑前，不进入实现。

## 建议的 Plan Mode 输出结构

```markdown
# Sage V7 Learning MVP 计划

## 1. 产品目标与非目标
## 2. 当前能力与可复用资产
## 3. 第一条端到端用户流程
## 4. 领域模型与状态机
## 5. 系统架构与数据流
## 6. V7 分阶段里程碑
## 7. V8 索引与知识图谱演进
## 8. Benchmark
## 9. 安全、隐私与失败恢复
## 10. 旧路线迁移表
## 11. 文档与面试叙事迁移
## 12. 开放问题与决策门
```

## 第一轮只问四个问题

1. V7 第一版数据源是否只支持“代码仓库 + Obsidian/Markdown”，把 PDF 和课程转录留到后续？
2. `LearningState` 提升至少需要哪类 evidence：测验、费曼解释、代码实验、项目产物、人工确认中的哪些组合？
3. 学习计划由用户显式创建，还是 Sage 可以主动提出但必须由用户确认？
4. V7 保持纯本地单机，还是保留一个最小远程访问路径？

一次只向用户确认一个决策，避免同时展开所有分支。
