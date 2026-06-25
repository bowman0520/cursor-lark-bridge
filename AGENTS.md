# AGENTS.md

本文件是 `cursor-lark-bridge` 子项目的唯一主规则文件。任何 agent（Codex / Claude / Gemini / Cursor 等）在本目录工作时，都以本文件为准。

## 项目定位

把飞书 / Lark 消息桥接到本地 **Cursor Agent CLI（`cursor-agent`）** 的机器人。fork 自 `lark-channel-bridge`（原版桥接 `claude`），把后端 agent 换成了 `cursor-agent`，飞书侧管道逻辑保持不变。

放置位置：`~/Projects/bridges/`（桥接类项目）。

## 源码与运行时边界

- **本项目是「只有打包产物、没有 `src/`」的 bundle。** 所有逻辑在 `dist/cli.js`（单文件 bundle，约 6000 行）。改动直接打补丁到 `dist/cli.js`，没有可重新构建的源码。
- `bin/cursor-lark-bridge.mjs` 只是入口，`import '../dist/cli.js'`。
- 运行时数据目录：`~/.cursor-lark/`（与 Claude 版 `~/.lark-channel/` 完全隔离，可同机并存）。
- 守护进程服务名：`cursor-lark-bridge.bot`（launchd label `ai.cursor-lark-bridge.bot`），与 Claude 版隔离。

## Agent 回复约定

- 用户叫 **Bowman**；通过 bridge 会话回复时，**每次都要带上 Bowman 的名字**（开头称呼、文中点名或结尾署名均可）。

## Bridge 斜杠命令

- `/model`、`/usage`、`/cd`、`/new`、`/status` 等由 bridge 本地处理，不交给 agent。
- `/usage` 调用本机 `cursor-usage` 查询 Cursor 双池额度（Auto+Composer / API）。

## 相对原版（Claude）已改动的核心点

集中在 `dist/cli.js` 三处，改动时优先只碰这些：

1. `CursorAdapter`（原 `ClaudeAdapter`）：`spawn("cursor-agent", ["-p","--output-format","stream-json","--force",...])`；不支持 `--append-system-prompt`，改为会话首条消息前拼 `BRIDGE_SYSTEM_PROMPT`；不支持 base64 stdin 图片，改为把图片路径写进 prompt；`--resume <id>`、`--model`、`--workspace <cwd>`。
2. `translateEvent`：适配 Cursor 的 stream-json——`thinking` 是顶层事件；工具调用是 `tool_call` 的 `started/completed`（工具名从 `xxxToolCall` 的 key 推导，结果在 `tool_call.xxxToolCall.result.success|error`）；`result.usage` 用驼峰 `inputTokens/outputTokens`，无 cost。
3. 卡片回调 marker `__claude_cb` → `__cursor_cb`（常量 `CLAUDE_CALLBACK_MARKER` 的值 + 系统提示文案）。

数据目录 / 服务名 / 命令名 / 品牌文案已替换；secrets 与 lark-cli 集成的内部 `source` 标识符**保持原值不要改**，否则会破坏密钥库绑定。

## 常用命令

```bash
node bin/cursor-lark-bridge.mjs run        # 前台运行
node bin/cursor-lark-bridge.mjs --help     # 全部命令
node bin/cursor-lark-bridge.mjs start      # 安装并以系统守护进程启动
node bin/cursor-lark-bridge.mjs status     # 守护进程状态
```

前置：`cursor-agent` 已安装并登录（`cursor-agent status` 显示已登录）；Node >= 20。

## 测试方式

- 没有单测套件（bundle-only）。验证 `translateEvent` 时，抽出函数喂真实 stream-json 样本跑。
- 抓 Cursor 真实输出格式：`cursor-agent -p --output-format stream-json --force "<prompt>"`。
- 改完至少跑通 `node bin/cursor-lark-bridge.mjs --help` 确认 bundle 能解析。

## 关键配置路径

- `~/.cursor-lark/config.json`：应用凭据（600）
- `~/.cursor-lark/sessions.json`：每聊天 session id + cwd
- `~/.cursor-lark/workspaces.json`：命名工作区
- `~/.cursor-lark/logs/YYYY-MM-DD.log`：运行日志（JSONL）

## 迁移安全

- 移动 / 改名本项目前，先检查 launchd plist（`~/Library/LaunchAgents/ai.cursor-lark-bridge.bot.plist`）、守护进程注册、绝对路径引用。
- 不要把数据目录改回 `~/.lark-channel`，会和 Claude 版抢配置和守护进程。
