# Schema

## 三层结构

1. Raw Sources：不可变来源快照，Agent 禁止覆盖已有 revision。
2. Wiki：Agent 维护、用户阅读和审核的 Markdown 知识页面。
3. Schema：结构、命名、引用、审核和发布规则。

## 页面 frontmatter

```yaml
id: page_uuid
type: source | project | concept | decision | query | learning
title: 页面标题
status: draft | verified | superseded
visibility: private | public_candidate | published
sources:
  - source_id: source_uuid
    revision: sha256:...
```

## 写入约束

- 所有 Wiki 新建、修改和回滚必须先产生 proposal 与 diff。
- 模型生成页面默认 `draft`，不能自动标记为 `verified`。
- 回滚通过新 revision 恢复旧内容，不重写或删除 Git 历史。
- `index.md` 是内容目录；`log.md` 是 append-only 操作记录。
- `[[wikilink]]` 指向仓库内页面，不使用无法稳定解析的临时编号。
- 禁止写入密钥、Cookie、私有会话原文、未筛选个人信息和第三方受限内容。
- 公开发布必须从 `public_candidate` 经过独立扫描和审批。

## 首阶段摄取

V7.2-P2.1 只摄取白名单 Markdown 单文件，先生成来源投影页面。跨来源综合、概念页更新、Lint、RAG 和飞书 Source Adapter 在状态机稳定后分阶段加入。
