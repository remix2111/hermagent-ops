---
name: hermes-ops
description: Hermes Agent 运维技能合集 — 建 Profile 铁律、CLI 卡死恢复、TUI 命令调试。
version: 1.0.0
user-invocable: true
tags: [hermes, ops, profile, tmux, tui, debugging]
---

# Hermes 运维技能合集

本技能合并了 Hermes Agent 运维中最常用的三个能力：创建 Profile 的铁律模板、CLI 卡死的 tmux 恢复方案、以及 TUI 斜杠命令的调试流程。

---

## 一、建 Profile 铁律

> 来源：`hermes-profile-iron-rules`

从 30+ 次事故中提炼的三条不可违反的规则。每个新 Profile 的 SOUL.md 必须包含。

### 什么时候用

- 创建新 Profile（`hermes profile create`）
- Profile 的 SOUL.md 不包含这三条
- Agent 总是跳过计划直接改代码
- Agent 频繁全量重写导致功能崩溃

### 模板

直接复制到 Profile 的 SOUL.md 中：

```markdown
## 🔴 三条铁律

**1. 先计划，后动手。** 做任何改动前，先列出计划给用户过目。他说「做」再执行。不自己闷头干。

**2. 定点修复，不全量重写。** 改一个按钮只动那个按钮的代码。改一段文案只替换那段文案。永远不要重写整个文件。

**3. 改前备份，改完验证。** 任何改动前先 cp 备份。改完立刻检查页面/功能是否正常。
```

### 🧠 Profile Memory Bank（受 Agent-Zero 启发）

每个 Profile 必须设置 memory-bank/ 目录，用文件做结构化工作记忆：

```bash
mkdir -p ~/.hermes/profiles/<name>/memory-bank/
touch ~/.hermes/profiles/<name>/memory-bank/{whoami,progress,learnings}.md
```

三文件职责：

| 文件 | 放什么 | 更新频率 |
|------|--------|----------|
| `whoami.md` | 身份定义、核心任务、数据管道、风险纪律、自检清单 | 几乎不改 |
| `progress.md` | 上次做了什么、当前问题（带优先级🔴🟡🟢）、下次要做、参考路径 | 每次任务后 |
| `learnings.md` | 重复出现的模式、被纠正的错误、工具使用坑、策略决策记录 | 每次新发现 |

Default profile 的特殊路径：`~/.hermes/memory-bank/`（不在 profiles/default/ 下）

### 为什么只有三条

加多了被忽略。这三条是所有事故的根因——改了再问、全量重写、改完不管。其他规则（语言、风格、职责）每个 Profile 自己加。

### 已验证 Profile

| Profile | 加入时间 | 出事次数（加入后） |
|---------|---------|------------------|
| default | 2026-06-08 | 0 |
| singxai | 2026-06-07 | 0 |
| momen | 2026-06-08 | 0 |

---

## 二、CLI 卡死恢复

> 来源：`hermes-cli-tmux-recovery`

### 问题表现

- 关闭终端窗口后，`hermes chat` 会话报 `OSError [Errno 5] Input/output error`
- stdout 断开导致 Hermes 进程崩溃

### 根因

`hermes chat` 直接绑定到终端 stdout。关闭终端时 stdout 文件描述符失效，写入时报错。Systemd 跑独立 gateway 不可行，因为 chat 模式会自动拉起 gateway 导致冲突。

### 正确方案：tmux

```bash
# 创建会话
tmux new -s hermes

# 在 tmux 里运行
hermes chat

# 断开终端（按 Ctrl+B, D 或直接关终端）
# 会话继续运行

# 重新连接
tmux attach -t hermes
```

### 要点

- 关闭终端后，tmux 会话在后台保持运行
- 重连只需 `tmux attach -t hermes`
- 不需要 systemd 或其他方案
- 重启电脑后 tmux 会话消失，需要重新 `tmux new` + `hermes chat`

---

## 三、TUI 命令调试

> 来源：`debugging-hermes-tui-commands`

### 概述

Hermes 斜杠命令跨三层——Python 命令注册表、tui_gateway JSON-RPC 桥接、以及 Ink/TypeScript 前端。当命令出问题时（自动补全缺失、CLI 能用但 TUI 不行、配置已保存但 UI 不更新），bug 几乎总是某一层与另外两层不同步。

### 架构总览

```
Python 后端 (hermes_cli/commands.py)     ← 权威 COMMAND_REGISTRY
       │
       ▼
TUI 网关 (tui_gateway/server.py)         ← slash.exec / command.dispatch
       │
       ▼
TUI 前端 (ui-tui/src/app/slash/)         ← 本地处理器 + 回退
```

Python `COMMAND_REGISTRY` 是以下各项的唯一真相来源：CLI 分发、gateway 帮助、Telegram BotCommand 菜单、Slack 子命令映射、发送给 Ink 的自动补全数据。

### 调查步骤

1. **检查命令是否在 TUI 前端存在：**
   ```bash
   search_files --pattern "/commandname" --file_glob "*.ts" --path ui-tui/
   search_files --pattern "/commandname" --file_glob "*.tsx" --path ui-tui/
   ```

2. **检查 TUI 命令定义：**
   ```bash
   read_file ui-tui/src/app/slash/commands/core.ts
   # 如果不在 core.ts，搜索命令文件：
   search_files --pattern "commandname" --path ui-tui/src/app/slash/commands --target files
   ```

3. **检查 Python 后端：**
   ```bash
   search_files --pattern "CommandDef" --file_glob "*.py" --path hermes_cli/
   search_files --pattern "commandname" --path hermes_cli/commands.py --context 3
   ```

4. **检查 gateway 实现：**
   ```bash
   search_files --pattern "complete.slash|slash.exec" --path tui_gateway/
   ```

### 修复：缺少命令自动补全

如果命令在 TUI 中存在但不显示在自动补全中：

1. 在 `hermes_cli/commands.py` 的 `COMMAND_REGISTRY` 中添加 `CommandDef` 条目：
   ```python
   CommandDef("commandname", "命令描述", "Session",
              cli_only=True, aliases=("alias",),
              args_hint="[arg1|arg2|arg3]",
              subcommands=("arg1", "arg2", "arg3")),
   ```

2. 谨慎选择 `cli_only` vs gateway 可用性：
   - `cli_only=True` — 仅在交互式 CLI/TUI 中可用
   - `gateway_only=True` — 仅在消息平台中可用
   - 两者都不设 — 随处可用
   - `gateway_config_gate="display.foo"` — gateway 中受配置门控

3. 确保 `subcommands` 与 TUI 显示的 Tab 补全选项匹配。

4. 如果命令在服务端运行，在 `cli.py` 的 `HermesCLI.process_command()` 中添加处理器：
   ```python
   elif canonical == "commandname":
       self._handle_commandname(cmd_original)
   ```

5. 对于 gateway 可用的命令，在 `gateway/run.py` 中添加处理器：
   ```python
   if canonical == "commandname":
       return await self._handle_commandname(event)
   ```

### 常见问题

1. **命令出现在 TUI 中但不在自动补全中。** 命令在 TUI 代码库中定义了，但 `hermes_cli/commands.py` 的 `COMMAND_REGISTRY` 中缺失。自动补全数据从 Python 端下发。

2. **命令出现在自动补全中但不能用。** 检查 `tui_gateway/server.py` 中的命令处理器和 `ui-tui/src/app/createSlashHandler.ts` 中的前端处理器。如果命令是 Ink 本地命令，必须在 `app.tsx` 内建分支中处理；否则通过 `slash.exec` 回退到 Python 处理器。

3. **命令在 CLI 和 TUI 中行为不一致。** 命令可能有不同实现。同时检查 `cli.py::process_command` 和 TUI 的本地处理器。本地 TUI 处理器优先于 gateway 分发。

4. **命令保存了配置但实时不生效。** 对于 TUI 本地命令，仅更新 `config.set` 不够。还需要立即 patch 相关 nanostore 状态（通常是 `patchUiState(...)`）并将新状态通过渲染组件传递。

5. **Gateway 分发静默忽略命令。** Gateway 只分发它认识的命令。检查 `GATEWAY_KNOWN_COMMANDS`（自动从 `COMMAND_REGISTRY` 派生）是否包含规范名称。如果命令是 `cli_only` 且带 `gateway_config_gate`，验证门控配置值为真。

### 调试技巧

当表层检查无法定位 bug 时：

- **Python 端挂起或异常：** 使用 `python-debugpy` 技能在 `_SlashWorker.exec` 或命令处理器中打断点。`remote-pdb` 设在处理器入口是最快路径。
- **Ink 端未响应：** 使用 `node-inspect-debugger` 技能在 `app.tsx` 的斜杠分发或本地命令分支打断点。`sb('dist/app.js', <line>)` 在 `npm run build` 之后。
- **注册表不匹配 / 不确定哪端有问题：** 将规范 `COMMAND_REGISTRY` 条目与 TUI 本地命令列表逐项对比。

### 注意事项

- 别忘了在 `CommandDef` 中设置正确类别（如 "Session"、"Configuration"、"Tools & Skills"、"Info"、"Exit"）
- 确保别名正确注册在 `aliases` 元组中——无需修改其他文件，所有下游（Telegram 菜单、Slack 映射、自动补全、帮助）都从它派生
- 带子命令的命令，确保 `CommandDef` 中的 `subcommands` 元组与 TUI 代码中一致
- `cli_only=True` 的命令不会在 gateway/消息平台中工作——除非添加 `gateway_config_gate` 且门控为真
- 添加实时 UI 状态后，搜索旧 prop/helper 的所有消费者，将新状态贯穿所有渲染路径，不仅是活跃流式传输路径
- 测试前重建 TUI（`npm --prefix ui-tui run build`）——tsx watch 模式可能首次启动延迟

### 验证

修复后：

```bash
# 1. 重建 TUI
cd /home/bb/hermes-agent && npm --prefix ui-tui run build

# 2. 运行 TUI 测试命令
hermes --tui

# 3. 输入 / 验证命令出现在自动补全中，附带预期描述和参数提示

# 4. 执行命令确认：
#    - 预期行为触发
#    - 持久化配置正确更新（read_file ~/.hermes/config.yaml）
#    - 实时 UI 状态立即反映更改（而不仅仅是重启后）

# 5. 如果命令也 gateway 可用，从至少一个消息平台测试（或运行 gateway 测试）
```
