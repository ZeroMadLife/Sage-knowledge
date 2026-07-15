---
id: page_b338c7d5e9c5bbe9b6dbe844a4924f57
type: source
title: "飞书 cc-connect 与 Claude Code 链路收口"
status: draft
visibility: private
sources:
  - source_id: src_b338c7d5e9c5bbe9b6dbe844a4924f57
    kind: obsidian
    path: "44-飞书cc-connect与ClaudeCode链路收口.md"
    revision: sha256:cfaf22279dd3a63647ea1186a1898cd3251454b394cfc99e57711cb4fa2f5a26
raw_snapshot: raw/sources/obsidian/cf/cfaf22279dd3a63647ea1186a1898cd3251454b394cfc99e57711cb4fa2f5a26.md
parser_id: sage.markdown
parser_version: 1.0.0
parsed_document_id: pdoc_8ccebf78571a43215c540b2edb64fe56
---

# 飞书 cc-connect 与 Claude Code 链路收口

> 本页是可审核的来源投影。后续 LLM 综合必须继续保留来源 revision。

## 来源内容

---
title: 飞书 cc-connect 与 Claude Code 链路收口
type: operational-retrospective
project: Sage
source_project: /Users/zeromadlife/Desktop/tour-agent
source_branch: dev/sage-v6
source_commit: a3672c7
verified_at: 2026-07-15
tags: [sage, feishu, cc-connect, claude-code, operations, security]
---

# 飞书 cc-connect 与 Claude Code 链路收口

## 阶段结论

Sage 的第一条飞书开发链路已跑通：目标群中的 `@机器人` 消息经飞书 WebSocket
长连接进入本机 `cc-connect`，再启动 Claude Code 读取
`/Users/zeromadlife/Desktop/tour-agent`，并把结果回复到原群。

```text
飞书群
  -> 企业自建机器人
  -> cc-connect 1.4.1
  -> Claude Code 2.1.185
  -> Sage dev/sage-v6
  -> 飞书群回复
```

## 配置结果

- cc-connect 项目名：`sage`；
- Agent：`claudecode`；
- 权限模式：`default`；
- 工作目录：`/Users/zeromadlife/Desktop/tour-agent`；
- 飞书应用版本：`1.0.1`，状态为已发布；
- 用户级配置：`~/.cc-connect/config.toml`，权限 `0600`；
- 临时后台任务：`com.zeromadlife.cc-connect-sage`；
- 管理页：`http://localhost:9820`。

稳定版 `1.4.1` 的 Web UI 飞书向导只支持二维码新建应用。现有应用通过
`cc-connect feishu bind` 绑定，App Secret 由 macOS 隐藏输入对话框采集，没有写入
Sage 仓库、设计文档、shell 历史或验收日志。

## 验证证据

- 飞书凭据校验成功，cc-connect 能识别机器人身份；
- WebSocket 与飞书网关建立长连接；
- 群聊 `im.message.receive_v1` 事件成功进入 `sage` 项目；
- Claude Code 新建 session，首轮只读任务调用 1 个工具；
- 约 13 秒完成，回复长度约 1,576 字符，并发送回原群；
- 测试前后 `git status --short` 一致：仅保留既有 `test-xxx.txt` 与 `tmp/`；
- `git diff --check` 通过；本阶段没有修改产品代码，因此未重复运行产品测试和构建。

## Git 与集成结论

接入设计以 `0f3bc1d` 提交，实际验证结果以 `a3672c7` 更新在
`dev/sage-v6`。这是集成分支上的操作设计文档，没有独立功能分支或 worktree
需要合并、清理。Sage 产品代码未变化，当前结论为：链路可用，无产品代码合并动作。

## 安全复盘

已关闭边界：

- Claude Code 保持 `default`，没有启用 `bypassPermissions`；
- `admin_from` 为空，`/shell`、`/restart`、`/upgrade` 等特权命令被阻止；
- 首轮提示明确只读，Git 状态证明没有产生新改动；
- App Secret 只保存在权限为 `0600` 的用户配置中。

仍需关闭的风险：

- `allow_from` 为空，目标群内所有成员当前都能触发普通任务；
- `card.action.trigger` 回调尚未单独验收，后续写操作的卡片审批可能超时；
- 当前 `launchctl submit` 任务只保证本次登录会话在线，不跨注销或重启；
- App Secret 曾进入聊天记录，用户选择暂不轮换；
- Codex CLI 的 macOS ARM64 平台二进制缺失，尚不能作为第二 provider；
- 部署命令、生产凭据、回滚与审批策略均未开放。

## 下一阶段边界

1. 将 `allow_from` 收紧到明确的飞书用户或成员集合。
2. 验证 `card.action.trigger`，再尝试受控写文件和测试命令。
3. 为群内开发任务定义 per-task worktree、测试、diff、提交和合并审批流程。
4. 单独设计部署权限、生产凭据、健康检查和回滚，不与普通开发权限共用。
5. 需要长期运行时再创建可审计的持久 LaunchAgent，并补充重启恢复验证。
6. 修复 Codex CLI 后，再评估 `exec` 与 `app_server` 后端。
