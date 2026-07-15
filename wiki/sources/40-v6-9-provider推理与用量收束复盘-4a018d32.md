---
id: page_452e5ce135040ddc8828716a4a018d32
type: source
title: "V6.9 Provider、推理与用量收束复盘"
status: draft
visibility: private
sources:
  - source_id: src_452e5ce135040ddc8828716a4a018d32
    kind: obsidian
    path: "40-V6.9-Provider推理与用量收束复盘.md"
    revision: sha256:1f3fcfe8cdca6a4092ef83c4890a678c69a1d8b6e2a02cc22ee0402920560375
raw_snapshot: raw/sources/obsidian/1f/1f3fcfe8cdca6a4092ef83c4890a678c69a1d8b6e2a02cc22ee0402920560375.md
parser_id: sage.markdown
parser_version: 1.0.0
parsed_document_id: pdoc_688a966605ff834dcc41717a5a5a95a3
---

# V6.9 Provider、推理与用量收束复盘

> 本页是可审核的来源投影。后续 LLM 综合必须继续保留来源 revision。

## 来源内容

---
date: 2026-07-14
project: sage
source_project: /Users/zeromadlife/Desktop/tour-agent
source_branch: dev/sage-v6
source_commit: 2016852
status: integrated
tags: [sage, v6.9, provider, reasoning, usage, frontend, security, learning]
---

# V6.9 Provider、推理与用量收束复盘

> 本章记录 V6.9 最后一轮 Provider 配置、真实 reasoning、真实用量和聊天门面的实际交付。设计稿和计划不计入功能完成度，以下结论以 `2016852 feat(sage-v6.9): close provider reasoning and usage`、自动化测试和真实浏览器联调为准。

## 实际交付

### 1. `.sage` 成为项目级配置入口

新增 `.sage/settings.json` 严格 JSON 配置，旧 `config/coding_models.toml` 在新文件缺失时只读回退。网页只读写非敏感目录信息，Provider 密钥只允许声明环境变量名 `api_key_env`，不会接收或回显 Key。

服务端对未知字段、重复 Provider/模型、非法 URL、非法环境变量名、错误 context 和不兼容 reasoning 描述执行严格拒绝。写入采用临时文件、`fsync`、`0600` 和 `os.replace`；项目配置与 usage 数据库在读取和连接时拒绝 symlink。

项目指令加载顺序固定为：

1. `.sage/SAGE.md`
2. 根目录 `SAGE.md`
3. 根目录 `AGENTS.md`

### 2. reasoning 是真实请求契约，不是伪开关

会话持久化 `reasoning_mode`，模型切换时会对不兼容档位回退 `off`，活动 run 或上下文压缩期间拒绝切换。后台 worker 继承当前模型与 reasoning。

- OpenAI-compatible 模型只有显式声明 `openai_reasoning_effort` 时才传 `reasoning_effort`；
- Anthropic 模型只有显式声明 `anthropic_thinking_budget` 时才传 `thinking` budget；
- 当前 DeepSeek Flash/Pro 的旧目录没有声明 reasoning，因此界面正确显示“不支持”，不会伪造思考开关。

Provider 的 thinking block 采用公开文本类型白名单过滤，不进入 delta、transcript、timeline、run trace 或最终 Markdown。聊天中的“正在思考”只显示公开阶段和经过秒数。

### 3. 用量来自 Provider 元数据

新增 `.sage/usage.sqlite3`，按 `run_id:model_attempt` 幂等记录 Provider 返回的 input/output/cache/total token 元数据。账本不保存 prompt、回答、工具结果、密钥、header 或完整 Provider 响应。

`GET /api/v1/coding/usage?range=7d|30d|90d|365d` 返回日期、模型和会话聚合。Provider 未返回的字段保持 `null`；未配置价格表时费用显示 `--`，不伪造 `0` 或估算值。空用量查询不会为了展示空态而创建 `.sage` 目录。

### 4. 聊天和设置门面完成收束

Composer 现在采用“输入区 + 右上 context budget + 底部 command rail”，保留权限、模型、reasoning、发送/停止和 Provider 设置入口。运行态使用 Sage 原创金发 Q 版 CSS 头像、阶段、秒数和 reduced-motion 兼容流光。

设置中心新增 Provider 和用量页面：

- Provider 页面显示协议、base URL、模型数量、环境变量名和“已配置/未配置”，没有密码输入框；
- 本地模式可编辑 `.sage/settings.json`，部署托管模式只读；
- 用量页面支持 7/30/90/365 天、真实 token、缓存命中率、模型分布、空态和未知费用；
- Provider 与用量拥有独立错误状态，快速切换范围时旧响应不能覆盖新范围。

## 真实浏览器验收

隔离服务：前端 `http://127.0.0.1:5210`，后端 `http://127.0.0.1:8210`。没有混用 `5173/8000` 作为隔离分支证据。

实际执行流程：

```text
用户消息
  -> 正在思考（公开阶段、秒数、流光）
  -> run_shell 审批
  -> 工具参数与结果展开/收起
  -> Markdown 最终回答
  -> JSON / Python 代码高亮
  -> Provider / 用量设置
```

`run_shell` 实际执行 `pwd && printf 'sage-shell-ok\n'`，返回 `exit_code: 0` 和正确 worktree 路径。Provider 返回的该会话用量在页面聚合为 2 次模型请求，证明不是前端估算。桌面与 `390 × 844` 均无横向溢出；移动端 `documentScrollWidth = 390`，控制台没有相关 error/warn。

## 最终验证证据

隔离分支最终 HEAD：

- 后端 coding/API/LLM 全集：`739 passed`；
- 前端全量：`24 files / 246 tests passed`；
- 前端 production build：通过；
- `ruff`、`mypy`（10 个受影响源码）、`compileall`、`git diff --check`：通过；
- 一次并行高负载运行中 router `beforeEach` 超过 10 秒；无竞争环境整套复跑 `246 passed`，判定为资源型 flake，不是行为断言失败。

fast-forward 合入 `dev/sage-v6@2016852` 后，集成 worktree 复测：

- 后端受影响集：`56 passed`；
- 前端核心集：`4 files / 97 tests passed`；
- production build、`git diff --check`：通过；
- 集成 worktree 干净。

## 关闭的风险

- 应用构造不再提前创建 workspace 或 `.sage`；
- Provider 请求体中的未知敏感值不会由 FastAPI 默认校验错误回显；
- settings 与 usage 文件读取/连接拒绝 symlink；
- thinking block 不会作为单个对象或列表内容进入用户可见正文；
- usage 流式尾包、重连或重放不会重复累计；
- Provider 与 usage 的并行请求错误不会相互覆盖；
- 用户消息、思考态、工具过程和最终回答的顺序已用真实运行验证。

## 遗留边界与下一阶段

- 本轮没有生成 PNG 头像：当前会话缺少可调用的内置生图工具，且未获准切换到需要 `OPENAI_API_KEY` 的 CLI fallback。当前交付是原创 CSS 金发 Q 版角色，不复制用户参考角色。
- 当前真实 DeepSeek 目录未声明 reasoning，所以只验证了 `off` 和真实 usage；OpenAI/Anthropic reasoning 参数由单元契约验证，尚未使用对应远端凭据做在线 smoke。
- 生产构建仍有既有的 `CodingView` 大于 500 kB chunk 警告；代码分包属于后续性能治理。
- V7 的多租户、云密钥托管、服务器部署与 CI/CD 没有混入 V6.9。后续只从已验证的 `dev/sage-v6@2016852` 向 V7 线同步，不反向带入未完成的 V7 控制面。

## V7 同步结果

2026-07-14 已将已验证的 `dev/sage-v6@2016852` 合并到独立 V7 线：

- 目标分支：`codex/feat-v7-control-plane`；
- merge commit：`17d6f19 merge(sage-v7): integrate validated v6.9 closure`；
- 共同祖先：`7f73e93`；
- 唯一冲突：`api/schemas.py` 的 Pydantic import。最终同时保留 V7 云认证的 `field_validator` 和 V6.9 Provider 严格 schema 所需的 `ConfigDict`，没有选择任一整文件版本；
- V7 合并后验证：Cloud auth/GitHub OAuth/workspace 加 V6.9 Provider/usage/engine/LLM 集合 `88 passed`，前端全量 `24 files / 246 tests passed`，production build、`ruff`、`mypy`、`compileall`、`git diff --check` 均通过；
- V7 worktree 之前没有 `node_modules`，验证时以既有 `package-lock.json` 执行 `npm ci`，未改动依赖清单。

V7 的认证、GitHub OAuth、云控制面代码未被回退；此同步只把已经通过 V6.9 门禁的 coding runtime 和聊天门面带入后续服务器部署/CICD 线。

## Git 收口结论

- 需求完成度：Provider 非敏感配置、真实 reasoning 契约、真实 usage、Composer/运行态/设置页和真实 shell/Markdown 流程均已交付。
- 集成判定：**已 fast-forward 合入 `dev/sage-v6@2016852`，集成验证通过。**
- 短期分支判定：`codex/feat-v6-9-provider-usage-closure` 已被集成分支包含且干净，可在停止隔离服务后删除 worktree 和本地短期分支。
- V7 边界：**已同步到 `codex/feat-v7-control-plane@17d6f19`**，不得反向把尚未完成的 V7 认证/云控改动带回 `dev/sage-v6`。

## 关联阅读

- [[20-Hermes-Studio前端对标研究]]
- [[36-V6.9聊天主视图收线复盘]]
- [[37-V6.9-Hermes门面与工具恢复收口]]
- [[39-V6.9工作台运行体验收口]]
