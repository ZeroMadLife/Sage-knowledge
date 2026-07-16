---
id: page_f7de7a5da892998fafdc5ae4a98ee248
type: source
title: "V7.4 Knowledge Workspace UI 设计"
status: draft
visibility: private
sources:
  - source_id: src_f7de7a5da892998fafdc5ae4a98ee248
    kind: obsidian
    path: "64-V7.4-Knowledge-Workspace-UI设计.md"
    revision: sha256:9ba91ae5afd1214284d18344c1e4e9f09e1882a5715136ceab3560e324f96fa7
raw_snapshot: raw/sources/obsidian/9b/9ba91ae5afd1214284d18344c1e4e9f09e1882a5715136ceab3560e324f96fa7.md
parser_id: sage.markdown
parser_version: 1.0.0
parsed_document_id: pdoc_c798fe6a68ffb7c6fdded741a7020235
---

# V7.4 Knowledge Workspace UI 设计

> 本页是可审核的来源投影。后续 LLM 综合必须继续保留来源 revision。

## 来源内容

# V7.4 Knowledge Workspace UI 设计

## 版本证据

- Sage source commit：`a86516e1e081dc1a35214128fe1086b95c5de7e4`
- 开发分支：`dev/sage-v7`
- 完整设计书：`docs/plans/16-SAGE-V7.4-KNOWLEDGE-WORKSPACE-UI.md`
- 本轮性质：产品/UI 设计，不包含 Agent Harness 实现

## 核心判断

Knowledge 页面不应该把“直接向知识库提问”作为中心能力。当前检索表单和 citation 列表适合验证 RAG 契约，但正式用户真正关心的是：

1. 能不能一次导入本地文件和文件夹；
2. 能不能连接 GitHub、间接同步 Obsidian；
3. Sage 是否会自动解析、形成 Wiki、更新索引；
4. 普通内容是否无需逐条审核；
5. 最终知识结构、来源和变化是否清楚可见。

因此产品职责重新划分：

- Knowledge Workspace：建库、同步、观察、Wiki 阅读和异常治理；
- 主对话/个人助手：使用知识完成任务；
- RAG：继续作为底层能力，不在 Knowledge 页面复制第二套 Chat。

## 最终用户流程

```text
导入文件/文件夹或连接 GitHub
  -> 显示发现、解析、Vision、Wiki、索引进度
  -> 普通内容自动完成
  -> 页面原位切换到 Wiki Overview
  -> 浏览主题、页面、来源、revision 和异常
  -> 需要时把选中 scope 交给主对话使用
```

页面中的“自动”必须来自真实持久任务状态，不能只是前端动画。刷新或切走后回来，用户仍能观察同一任务。

## 页面结构

桌面采用稳定三栏加底部 Activity Bar：

- 左侧 256px：知识空间、添加来源、全部知识、最近更新、自动沉淀、需处理和来源树；
- 中间自适应：Overview、Wiki、Graph、Activity 与异常页；
- 右侧 340px：来源、页面或图节点 Inspector；
- 底部 56px：发现、解析、视觉处理、Wiki 更新和索引状态。

平板使用 Inspector Drawer；手机端 Sidebar 和 Inspector 使用全屏 sheet。

## 四种核心状态

### 空知识库

只显示拖入文件、选择文件、选择文件夹、连接 GitHub 和 Obsidian 接入方式。不显示空图谱、RAG 输入框、内部 revision、审核列表或索引重建按钮。

### 正在构建

显示发现、解析、视觉处理、Wiki 更新和索引进度。单个失败不阻断整个任务，使用“125 项完成，3 项需要处理”的摘要，不让用户确认 128 次。

### 构建完成

默认展示 Overview：本次变化、主题、项目、最近 Wiki 更新、来源健康和异常数量。图谱有真实数据后才开放。

### 存在异常

异常只包括授权失效、文件损坏、需要 OCR、低置信、来源冲突、敏感信息和高风险删除/公开。普通成功项永远不进入审核队列。

## GitHub 与 Obsidian

推荐让私有 `Sage-knowledge` Git repository 成为 canonical Wiki，Obsidian 打开同一 repository 或其中的 Markdown 目录。两者是同一知识资产的不同入口，不是两套互相覆盖的数据库。

浏览器可以一次性选择文件夹，但关闭页面后无法持续读取任意本地 Vault。因此能力必须如实表达：

- V7.4：文件/文件夹一次性导入；
- V7.4：GitHub 授权后持续同步；
- 当前推荐：Obsidian 通过 Git 推送后由 GitHub 同步；
- V8：Local Companion 直接观察本地 Vault。

## 视觉方向

- 深色产品顶栏配中性浅色工作区；
- 黑白灰承载主体信息；
- green 表示 synced，blue 表示 processing，amber 表示 attention，coral 表示 blocked，violet 只用于图社区；
- 正文 15–16px，导航 13–14px，元数据不低于 12px；
- 用轨道、边界和留白组织页面，不做套娃卡片；
- 空状态到构建态、构建态到 Overview 在同一 Canvas 内过渡，避免整页闪烁。

## 组件拆分

当前 `KnowledgeView.vue` 已超过 600 行，下一步拆为：

- `KnowledgeWorkspaceView`；
- `KnowledgeSidebar` 与 `SourceTree`；
- `KnowledgeCanvas`；
- `KnowledgeEmptyState`；
- `KnowledgeBuildProgress`；
- `KnowledgeOverview`；
- `WikiReader`；
- `KnowledgeActivity`；
- `KnowledgeAttention`；
- `KnowledgeInspector`；
- `KnowledgeImportDialog`；
- `KnowledgeActivityBar`；
- 真实 Graph API 完成后再接 `KnowledgeGraph`。

Knowledge Store 只保存来源、Wiki、同步任务、选中对象和面板状态，不接管 Chat timeline、Agent run、Memory 或模型状态。

## 开发顺序

### UI-0：现有能力重新编排

移除 Knowledge 页的 RAG 调试台，复用现有 summary、pages、jobs、proposal 和 index API，完成三栏 Shell、Overview、Activity Bar、异常入口和 Page Inspector。

### UI-1：多文件/文件夹导入

实现拖拽、文件夹选择、导入预检、持久进度恢复、部分失败和完成后的原位过渡。

### UI-2：GitHub/Obsidian 来源体验

实现 repository、branch、path 选择，来源健康、立即同步、授权失效和 Obsidian Git 引导。

### UI-3：真实知识图谱

Graph API 完成后再实现 Canvas、筛选、布局、图例和 Inspector；没有真实数据时保持 Wiki Overview，不画模拟节点。

## 会话协作边界

本 Knowledge UI 会话不设计个人助手 Agent Harness，也不修改主 Chat Composer、Coding timeline、Agent Loop、Tool Executor、Memory 或 Dream。

另一个会话负责个人助手特化。本会话只定义“在对话中使用”所需的选中 scope，具体如何检索、推理和回答由主对话会话决定。

共享 Router、全局导航和 Design Token 最后由 Integration 提交统一处理，避免两个会话同时编辑共享文件。

## 收口结论

- 设计结论：`可进入 UI-0 实施`
- 代码实现：尚未开始，不能把设计稿描述成已交付页面
- 后端能力：现有任务、Wiki、policy 和 index 足够支撑 UI-0；多文件上传与 GitHub 持续同步需要后续数据接口
- Graph：仍未实现，不能在前端使用假数据
- Agent Harness：明确由另一个会话负责
