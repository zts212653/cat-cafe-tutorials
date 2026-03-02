---
feature_ids: []
topics: [lessons, homework]
doc_kind: note
created: 2026-02-26
---

# 第五课 课后作业：给猫装上嘴巴

> 读完第五课的故事后，是时候动手了。
>
> 这份作业让你搭建一个最小的 MCP 回传系统：猫能在执行任务的过程中，主动往聊天室发消息。

---

## 作业目标

搭建一个简化版的 MCP 回传系统：
1. 一个 HTTP 服务器，接收猫发来的消息
2. 一个 CLI 子进程调用，让猫通过 MCP 工具主动发消息
3. 观察"内心独白"和"主动发言"的区别

完成后你会得到：
- 一个能接收 callback 的 HTTP 服务端
- 一个通过 MCP 主动发消息的 AI 调用
- 对"CLI 输出 vs post_message"分离的直观理解

---

## 前置条件

- [ ] 完成第一课作业（能用 spawn 调用 Claude CLI）
- [ ] 安装了 Claude Code CLI
- [ ] 已登录 Claude CLI
- [ ] 了解 MCP 的基本概念（第零课）

---

## 提示词

把下面的提示词复制给一个新的 Claude 对话，让它帮你完成作业：

```
我想搭建一个最小的 MCP 回传系统，理解"猫主动说话"的机制。

## 背景

我在学习 AI Agent 协作系统。核心概念是：
- AI 通过 CLI 子进程执行任务
- CLI 内部的输出是 AI 的"内心独白"，默认不可见
- AI 可以通过 MCP 工具主动调用 HTTP callback，把消息发到聊天室
- 这样 AI 就有了"选择说什么"的自主权

## 要求

帮我创建三个文件：

### 1. callback-server.js — HTTP 回调服务器

一个简单的 Node.js HTTP 服务器（不用框架，用原生 http 模块），监听 3200 端口：

- POST /api/callbacks/post-message
  - 接收 JSON body: { invocationId, callbackToken, content }
  - 验证 invocationId 和 callbackToken 是否匹配预设值
  - 如果验证通过，把 content 打印到终端（模拟"消息出现在聊天室"）
  - 返回 { status: "ok" }
  - 如果验证失败，返回 401

- GET /api/callbacks/thread-context
  - 接收 query params: invocationId, callbackToken
  - 验证通过后，返回一段模拟的对话历史 JSON
  - 比如: { messages: [{ role: "user", content: "请写一首关于猫的诗" }] }

启动时生成一对 UUID 作为 invocationId 和 callbackToken，打印出来。

### 2. cat-cafe-mcp.js — 最小 MCP Server

一个 MCP Server（使用 @modelcontextprotocol/sdk），提供两个工具：

- cat_cafe_post_message(content: string)
  - 向 callback-server 的 POST /api/callbacks/post-message 发 HTTP 请求
  - 从环境变量读取 CAT_CAFE_API_URL, CAT_CAFE_INVOCATION_ID, CAT_CAFE_CALLBACK_TOKEN

- cat_cafe_get_context()
  - 向 callback-server 的 GET /api/callbacks/thread-context 发 HTTP 请求
  - 返回对话上下文

### 3. run-cat.js — 调用脚本

用 spawn 调用 claude CLI，动态挂载 MCP Server：

```bash
claude -p "你的任务是写一首关于猫的诗。
在开始写之前，先用 cat_cafe_get_context 获取上下文。
写完后，用 cat_cafe_post_message 把诗发到聊天室。
注意：你的思考过程不需要发送，只把最终的诗发到聊天室即可。" \
  --output-format stream-json \
  --verbose \
  --mcp-config '{"mcpServers":{"cat-cafe":{"command":"node","args":["cat-cafe-mcp.js"]}}}'
```

脚本需要：
- 设置环境变量 CAT_CAFE_API_URL, CAT_CAFE_INVOCATION_ID, CAT_CAFE_CALLBACK_TOKEN
- 这些值从 callback-server 启动时打印的值获取（可以硬编码或通过参数传入）
- 解析 Claude CLI 的 NDJSON 输出（参考第一课作业）

## 运行方式

终端 1：
```bash
node callback-server.js
# 输出：Server listening on :3200
# 输出：invocationId: xxx
# 输出：callbackToken: yyy
```

终端 2：
```bash
CAT_CAFE_API_URL=http://localhost:3200 \
CAT_CAFE_INVOCATION_ID=xxx \
CAT_CAFE_CALLBACK_TOKEN=yyy \
node run-cat.js
```

## 预期结果

- 终端 2 会显示 Claude CLI 的内部输出（"内心独白"）
- 终端 1 会收到并打印 Claude 通过 post_message 发送的诗（"主动发言"）
- 两个终端的输出是不同的！终端 2 是 AI 的全部思考过程，终端 1 只有 AI 选择公开的内容

## 技术要求

- callback-server.js: 纯原生 Node.js，不用框架
- cat-cafe-mcp.js: 使用 @modelcontextprotocol/sdk（需要 npm install）
- run-cat.js: 纯原生 Node.js

请帮我写这三个文件，并解释核心设计。
```

---

## 验收标准

### 基本验收

1. 启动 callback-server.js，看到 invocationId 和 callbackToken 被打印
2. 运行 run-cat.js，Claude 执行任务
3. **终端 1**（服务器）收到并打印了 Claude 发来的诗
4. **终端 2**（客户端）显示了 Claude 的完整思考过程
5. **两个终端的输出不同**——这就是"内心独白 vs 主动发言"的区别

### 关键观察

- 如果你把 callbackToken 改成错误的值，Claude 的 post_message 会收到 401——这就是认证在工作
- Claude 的 NDJSON 输出里会包含 MCP 工具调用的记录（你能看到它"决定"调用 post_message 的那一刻）

---

## 进阶挑战（可选）

### 挑战 1：Prompt 注入版

不使用 `--mcp-config`，而是把 HTTP callback 指令写进系统提示词（模拟 Codex/Gemini 的做法）：

```
你可以通过以下 curl 命令发送消息：
curl -X POST http://localhost:3200/api/callbacks/post-message \
  -H "Content-Type: application/json" \
  -d '{"invocationId":"xxx","callbackToken":"yyy","content":"你的消息"}'
```

观察：AI 真的会照着 prompt 里的 curl 命令去执行吗？

### 挑战 2：对比实验

同一个任务，分别用两种方式调用：
- 方式 A：通过 `--mcp-config` 挂载 MCP Server（Claude 原生方式）
- 方式 B：通过系统提示词注入 curl 命令（Codex/Gemini 的兜底方式）

对比两种方式的行为差异。哪种更可靠？哪种的调用格式更稳定？

### 挑战 3：隐私模式

修改 run-cat.js，让它**不打印 Claude CLI 的内部输出**。只在 callback-server 的终端上看到 Claude 主动发送的消息。

这就是"猫猫杀模式"——其他玩家只能看到猫主动说的话，看不到猫在想什么。

---

## 常见问题

### Q: MCP Server 报错 "Cannot find module @modelcontextprotocol/sdk"

A: 需要先安装依赖。在作业目录运行：`npm init -y && npm install @modelcontextprotocol/sdk`

### Q: Claude CLI 报错 "MCP server failed to start"

A: 检查 `--mcp-config` 里的路径是否正确。路径必须是绝对路径或相对于当前工作目录的路径。

### Q: callback-server 没收到任何请求

A: 可能 Claude 的沙箱阻止了 MCP Server 发出站 HTTP 请求。检查 Claude CLI 的权限设置。也可能是环境变量没正确传递——确认 MCP Server 的 `process.env` 里能读到三个变量。

### Q: 两个终端的输出看起来一样

A: 检查 run-cat.js 是否把 Claude 的全部 stdout 都打印了。"内心独白"是 NDJSON 流里的 assistant 事件。"主动发言"是 callback-server 收到的 HTTP 请求。两者应该是不同的内容。

---

## 下一步

完成这个作业后，你已经理解了 MCP 回传的核心机制：
- HTTP callback 服务端接收猫的主动消息
- MCP Server 作为桥梁，让猫能调用 callback 工具
- 认证机制保证只有正确的猫能发消息
- CLI 输出和 post_message 是两个独立的通道

第六课会讲解这些消息存到哪里——从内存到 Redis 的存储层演进，以及一次 95% 数据丢失的惊险故事。

---

*这份作业由布偶猫设计。提示词遵循"教程用提示词而非代码模板"的原则——让 AI 帮你生成代码，你来理解和验证。*
