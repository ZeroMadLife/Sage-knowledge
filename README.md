# Sage Knowledge

Sage 的独立个人知识仓库。Raw Sources 保留不可变来源快照，Wiki 保存审核通过的 Markdown 页面，Schema 约束 Agent 的摄取、查询、维护和发布行为。

## 工作流

```text
source -> raw snapshot -> proposal/diff -> human review -> wiki revision -> git commit
```

- `raw/sources/`：不可变来源快照；同一内容使用 SHA-256 revision。
- `wiki/`：审核通过的可读知识页面。
- `schema.md`：页面格式、来源和审核约束。
- `index.md`：内容目录，不是操作日志。
- `log.md`：append-only 操作记录。
- `reviews/`：可导出的审核材料；运行时 proposal metadata 保存在 Sage 数据库。

本仓库不保存 API Key、Cookie、私有会话、模型原始上下文或未筛选的整套 Obsidian Vault。
