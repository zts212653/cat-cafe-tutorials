---
feature_ids: []
topics: [lessons, homework]
doc_kind: note
created: 2026-02-26
---

# 第一课 课后作业：用 CLI 调用 Claude Agent

> 读完第一课的故事后，是时候动手了。
>
> 这份作业包含一个提示词，你可以直接喂给 Claude（或其他 AI），让它帮你从零搭建一个最小可运行的 CLI 调用示例。

---

## 作业目标

写一个 Node.js 脚本，能够：
1. 用 `spawn()` 调用 `claude` CLI
2. 解析 NDJSON 流式输出
3. 打印出 Claude 的回复

完成后你会得到：
- 一个能跑的 `minimal-claude.js`
- 对 CLI 调用方式的直观理解
- 为第二课（完整实现）打下基础

---

## 前置条件

在开始之前，确保你已经：

- [ ] 安装了 Node.js (v18+)
- [ ] 安装了 Claude Code CLI (`claude --version` 能跑)
- [ ] 已登录 Claude CLI (`claude` 能正常对话)

### 安装 Claude Code CLI

官方安装方式（npm 已废弃）：

**macOS / Linux**：
```bash
# 推荐方式
curl -fsSL https://claude.ai/install.sh | bash

# 或用 Homebrew
brew install --cask claude-code
```

**Windows**：
```powershell
# 推荐方式
irm https://claude.ai/install.ps1 | iex

# 或用 WinGet
winget install Anthropic.ClaudeCode
```

安装后运行 `claude` 进入交互模式，会引导你登录。

> 详见官方文档：https://github.com/anthropics/claude-code

---

## 提示词

把下面的提示词复制给一个新的 Claude 对话（或其他 AI），让它帮你完成作业：

```
我想写一个 Node.js 脚本，用 child_process.spawn() 调用 Claude CLI，并解析它的流式输出。

## 背景知识

Claude CLI 支持以下调用方式：
- `claude -p "你的问题"` — 非交互模式
- `--output-format stream-json` — 输出 NDJSON（每行一个 JSON）
- `--verbose` — 必须和 stream-json 一起用

输出格式示例：
{"type":"system","subtype":"init","session_id":"abc123"}
{"type":"assistant","message":{"content":[{"type":"text","text":"Hello!"}]}}
{"type":"result","subtype":"success","session_id":"abc123"}

## 要求

1. 创建一个 `minimal-claude.js` 文件
2. 使用 Node.js 原生的 `child_process.spawn()`
3. 使用 `readline` 模块逐行解析 stdout
4. 解析 JSON，提取 `assistant` 类型事件中的文本内容
5. 打印出 Claude 的回复
6. 处理进程退出

## 运行方式

node minimal-claude.js "你好，请用一句话介绍自己"

## 不需要

- 不需要 TypeScript
- 不需要任何 npm 依赖（纯原生 Node.js）
- 不需要错误重试、超时处理（保持简单）

请帮我写这个脚本，并解释关键部分的代码。
```

---

## 验收标准

运行你的脚本：

```bash
node minimal-claude.js "用一句话介绍自己"
```

如果看到 Claude 的回复被打印出来，作业就完成了！

---

## 进阶挑战（可选）

如果基础作业太简单，可以尝试：

### 挑战 1：支持 Codex CLI

修改脚本，让它也能调用 `codex exec`：

```bash
node minimal-codex.js "你好"
```

提示：Codex 的 NDJSON 格式不同，事件类型是 `thread.started`、`item.completed` 等。

### 挑战 2：统一接口

创建一个 `invoke(cli, prompt)` 函数，能同时支持 Claude 和 Codex：

```javascript
await invoke('claude', '你好');
await invoke('codex', '你好');
```

### 挑战 3：添加 Session 恢复

让脚本能记住 session，下次调用时用 `--resume` 继续对话。

---

## 常见问题

### Q: 运行报错 "claude: command not found"

A: Claude CLI 没安装或没加到 PATH。问 AI："帮我安装 Claude CLI"

### Q: 运行报错 "Please log in"

A: 还没登录。运行 `claude` 进入交互模式，会引导你登录。

### Q: JSON 解析报错

A: 可能遇到了不完整的行。检查是否正确使用了 `readline` 逐行读取。

### Q: 没有任何输出

A: 检查是否加了 `--verbose` 参数。`--output-format stream-json` 必须搭配 `--verbose`。

---

## 下一步

完成这个作业后，你已经理解了 CLI 调用的基本原理。

第二课会深入讲解：
- 完整的 `spawnCli()` 封装
- NDJSON 流的边界情况（粘包、不完整行）
- 超时处理、僵尸进程防护
- Session 管理

---

*这份作业由布偶猫设计。如果你用这个提示词成功跑起来了，欢迎分享你的体验！*
