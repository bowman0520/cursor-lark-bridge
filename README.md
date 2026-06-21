# cursor-lark-bridge

把飞书 / Lark 消息桥接到本地 **Cursor Agent CLI（`cursor-agent`）** 的轻量机器人。在终端跑一条命令、扫码绑定一个飞书自建应用，就能在聊天里直接和 Cursor Agent 对话——读截图、改代码，凡是你在终端能做的都能做。

> 本项目 fork 自 [`lark-channel-bridge`](https://www.npmjs.com/package/lark-channel-bridge)（原版桥接 Claude Code CLI），把后端 agent 从 `claude` 换成了 `cursor-agent`。飞书侧的全部能力（流式卡片、按会话隔离 session、`/ws` 多工作区、斜杠命令、守护进程、访问控制）保持不变。

## 它能做什么

- 把飞书 / Lark 消息（私聊直接发，或群里 `@bot`）转发给本地 `cursor-agent`，在你指定的工作目录里运行。
- **流式卡片**：Cursor 的文字和工具调用实时更新在同一张卡片上。
- **按会话隔离**：每个聊天有自己独立的 Cursor session，对话能续上。
- **抢占 + 合批**：新消息会打断正在跑的任务；连发的消息会合并成一次请求。
- **多工作区**：`/ws` 在多个项目目录之间切换。
- **图片和文件**：直接发给机器人，会以本地路径形式交给 Cursor，用读取工具查看。

## 前置要求

- Node.js **>= 20**
- Cursor CLI 已安装并登录：`curl https://cursor.com/install -fsS | bash`，然后 `cursor-agent login`
  - 验证：`cursor-agent status` 应显示已登录
- 一个飞书 / Lark **自建应用**（首次启动的扫码向导可以帮你创建）

## 运行

```bash
# 方式一：直接用 node 跑（项目目录内）
node bin/cursor-lark-bridge.mjs run

# 方式二：全局安装后用命令名
npm i -g .
cursor-lark-bridge run
```

首次运行检测到没有配置应用时，会打开**扫码向导**：终端里渲染二维码 → 用飞书 / Lark 扫码 → 选择或创建应用 → 凭据写入 `~/.cursor-lark/config.json`。

## 与 Claude 版的关键差异

| 项 | 说明 |
|---|---|
| 后端 agent | `cursor-agent` 代替 `claude`，启动参数用 `-p --output-format stream-json --force` |
| 系统提示 | `cursor-agent` 不支持 `--append-system-prompt`，bridge 在每个会话首条消息前拼上运行约定 |
| 卡片回调 marker | `__claude_cb` → `__cursor_cb` |
| 多模态图片 | 不走 base64 stdin，改为把图片路径写进 prompt 让 Cursor 用读取工具看 |
| 成本显示 | Cursor 不返回 USD 成本，相关字段留空 |
| 数据目录 | `~/.cursor-lark/`（与 Claude 版 `~/.lark-channel/` 隔离，可同机并存） |
| 服务名 | `cursor-lark-bridge.bot` / `ai.cursor-lark-bridge.bot`（与 Claude 版隔离） |

## 数据目录

| 路径 | 内容 |
|---|---|
| `~/.cursor-lark/config.json` | 应用凭据（App ID / Secret），权限 600 |
| `~/.cursor-lark/sessions.json` | 每个聊天的 Cursor session id + cwd |
| `~/.cursor-lark/workspaces.json` | 命名工作区映射 |
| `~/.cursor-lark/media/<chatId>/` | 下载的图片 / 文件，24h 后清理 |
| `~/.cursor-lark/logs/YYYY-MM-DD.log` | 结构化运行日志（JSONL），按天滚动 |

## 飞书内常用斜杠命令

`/new` `/reset` 清当前会话；`/cd <path>` 切目录；`/ws list|save|use|remove` 工作区；`/status` 状态；`/config` 偏好（含模型、回复样式、访问控制）；`/stop` 停止当前运行；`/help` 帮助。其它 `/xxx` 原样转发给 agent。

## 许可证

[MIT](./LICENSE)
