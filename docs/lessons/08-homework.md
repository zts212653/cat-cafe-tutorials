# 第八课课后作业：搭建最小 Session Chain

> **目标**：亲手体验 context 满了 → seal → 新 session 按需回忆旧 session 的全过程。
>
> **前置**：已读[第八课](./08-session-management.md)

---

## 核心练习：Session Chain 模拟器

### 你要搭建什么

一个最小的 Session Chain 系统，包含三个角色：

```
你的电脑
├── session-store.mjs       ← 内存 Session Chain 存储 + HTTP API
├── session-tools-mcp.js    ← MCP Server，让猫能搜索/查询旧 session
└── run-session-chain.mjs   ← 模拟 Session 1 满了 → 启动 Session 2 → 回忆
```

### 提示词

把以下提示词喂给你的 AI 助手：

```
帮我搭建一个最小的 Session Chain 模拟器，理解"context 满了怎么换命"。

## 背景

AI Agent 的 context window 有上限。满了以后有两种做法：
A) 压缩：把对话摘要化，信息有损
B) Session Chain：封存当前 session，开新 session，新猫按需搜索旧 session

我要体验方案 B。

## 要求

### 1. session-store.mjs — Session 存储 + HTTP API

一个 Node.js HTTP 服务器（原生 http 模块，不用框架），端口 3210：

数据模型：
- sessions: Map<sessionId, { id, seq, status, messages[], sealedAt? }>
- status: "active" | "sealed"

API 端点：
- POST /api/sessions — 创建新 session，返回 { id, seq }
- POST /api/sessions/:id/messages — 往 session 追加一条消息 { role, content }
- POST /api/sessions/:id/seal — 封存 session（status → sealed，记录 sealedAt）
- GET /api/sessions — 列出所有 session（id, seq, status, messageCount, sealedAt）
- GET /api/sessions/:id — 获取 session 详情（含完整 messages）
- GET /api/sessions/:id/digest — 返回 session 的 digest（自动从 messages 生成摘要）
  digest 格式：按角色分类统计消息数，列出所有 assistant 消息的前 80 字符，最后附"未完成事项"提取（从最后 3 条消息中找含"TODO"/"待做"/"下一步"的内容）
- GET /api/sessions/search?q=关键词 — 全文搜索所有 sealed session 的消息内容
  返回匹配的 { sessionId, seq, messageIndex, snippet（前后各 50 字符）}

启动时创建一个 Session 1 并预填充 8 条模拟对话（模拟一个开发任务的上下文）：
- user: "帮我实现一个用户注册功能"
- assistant: "好的，我来设计数据模型。User 表需要 id, email, passwordHash, createdAt 四个字段。选择 bcrypt 做密码哈希，因为它有内置的 salt 且计算成本可调。"
- user: "用什么框架？"
- assistant: "推荐 Fastify，原因：1. 比 Express 快 2 倍 2. 原生 TypeScript 支持 3. JSON Schema 验证内置。TODO: 后续需要加 rate limiting 防暴力破解。"
- user: "好的，先写 API 路由"
- assistant: "POST /api/register 路由已实现。校验规则：email 格式 + 密码至少 8 位。返回 201 + userId。下一步：需要加邮箱验证流程，暂时跳过。"
- user: "加个登录接口"
- assistant: "POST /api/login 已实现。用 JWT 做 token，过期时间 24h。待做：refresh token 机制还没做，当前是硬过期。"

然后将 Session 1 封存（seal）。

### 2. session-tools-mcp.js — MCP Server

使用 @modelcontextprotocol/sdk，提供四个工具：

- list_sessions() — 调 GET /api/sessions，返回所有 session 列表
- read_session(sessionId) — 调 GET /api/sessions/:id，返回完整消息
- get_digest(sessionId) — 调 GET /api/sessions/:id/digest，返回 digest 摘要
- search_sessions(query) — 调 GET /api/sessions/search?q=...，全文搜索

所有工具从环境变量 SESSION_STORE_URL 读取服务地址（默认 http://localhost:3210）。

### 3. run-session-chain.mjs — 模拟 Session Chain 交接

用 spawn 调用 claude CLI，模拟"Session 2 的猫"醒来后的行为：

系统提示词（关键！）：
"""
你是 Session 2 的布偶猫。你刚从一次 session 交接中醒来。

你知道的事实：
- 你是第 2 个 session，前面有 1 个已封存的 session
- 你有一套 MCP 工具可以查询旧 session 的内容
- 你不记得 Session 1 里发生了什么——但你可以查

你的任务：
1. 先用 list_sessions 看看有哪些旧 session
2. 用 get_digest 读取上一个 session 的摘要
3. 找出 Session 1 里提到但还没完成的事项（TODO / 待做 / 下一步）
4. 用 search_sessions 搜索一个你感兴趣的技术决策细节
5. 整理一份简短的"交接报告"，格式如下：

## 交接报告
### Session 1 做了什么
（摘要）
### 未完成事项
（列表）
### 我查到的关键决策
（你搜索到的细节）
### Session 2 的计划
（你打算先做什么）

注意：不要猜测 Session 1 的内容。一切信息必须来自工具查询。
"""

运行后：
- 解析 Claude CLI 的 NDJSON 输出
- 打印 Claude 调用了哪些 MCP 工具（工具名 + 参数）
- 打印最终的交接报告

## 运行方式

终端 1：
```bash
node session-store.mjs
# → Session 1 created with 8 messages (sealed)
# → Server listening on :3210
```

终端 2：
```bash
SESSION_STORE_URL=http://localhost:3210 node run-session-chain.mjs
# → [MCP] list_sessions → 1 session found
# → [MCP] get_digest(session_1) → ...
# → [MCP] search_sessions("bcrypt") → 1 hit
# → === 交接报告 ===
# → ...
```

## 技术要求

- session-store.mjs: 纯原生 Node.js，不用框架
- session-tools-mcp.js: 使用 @modelcontextprotocol/sdk
- run-session-chain.mjs: 纯原生 Node.js
- 注释用中文
```

### 验收标准

完成后你应该观察到：

1. Session 2 的猫**不记得** Session 1 的任何内容（它的 prompt 里没有）
2. 但它通过 MCP 工具**查到了** Session 1 做了什么
3. 它能准确找出未完成事项（refresh token、邮箱验证、rate limiting）
4. 它能通过 `search_sessions` 搜到具体的技术决策（为什么用 bcrypt、为什么选 Fastify）
5. 最终的交接报告是**有据可查**的，不是猜的

**关键体验**：Session 2 的猫面对一片空白的 context，靠搜索工具重建了对前世的理解。这就是"读旧 session = 搜代码"的直观感受。

---

## 进阶挑战 1：压缩 vs 交接 — 谁丢的信息多？

对比两种策略的信息保真度：

```
帮我设计一个对比实验，测试"压缩"和"Session Chain 交接"哪个丢的信息更多。

实验设计：
1. 准备一段包含 15 条消息的模拟对话（开发一个功能的完整过程），
   其中包含 5 个关键技术决策（每个决策有明确的 WHY）。

2. 方案 A — 压缩模式：
   把 15 条消息压缩成 3 条摘要（模拟 CLI 压缩），
   然后问 AI："之前为什么选择了 X 而不是 Y？"（针对 5 个决策各问一个）
   记录 AI 能准确回答几个。

3. 方案 B — Session Chain 模式：
   把 15 条消息存入 session-store，封存。
   新 session 的 AI 用 search_sessions + read_session 查询后回答同样的 5 个问题。
   记录 AI 能准确回答几个。

4. 对比：
   - 哪种方式回答准确率更高？
   - 哪种方式的 AI 更会说"我不确定"而不是编答案？
   - 哪种方式消耗的 token 更少？

用 Markdown 表格展示对比结果。
```

**预期发现**：方案 B 的准确率更高，因为原始信息完整保留；方案 A 的猫更容易"编造"一个看似合理但实际错误的 WHY。

---

## 进阶挑战 2：跨 Thread 污染复现

亲手重现茶话会夺魂 bug：

```
帮我搭建一个最小的跨 thread 污染演示。

1. 修改 session-store.mjs：
   - 加入 thread 概念（每个 session 属于一个 threadId）
   - 但 session 查找故意用 BUG 版本：只按 catId 匹配，不看 threadId

2. 模拟两个 thread：
   - Thread A "写代码"：session 里有 5 条关于 Phase 5 开发的消息
   - Thread B "茶话会"：session 里有 5 条关于哲学讨论的消息

3. 演示污染：
   - 在 Thread B 里调用猫，但因为 session 查找不带 threadId，
     猫 resume 了 Thread A 的 session
   - 猫在茶话会里突然开始讨论 Phase 5 😱

4. 修复：
   - session 查找加上 threadId 过滤
   - 再次在 Thread B 里调用猫，这次正常了

打印对比：
- [BUG] Thread B 的猫说了什么（应该会提到 Phase 5）
- [FIX] Thread B 的猫说了什么（应该只聊哲学）
```

**关键体验**：亲眼看到"少了一个字段"如何导致灵魂串线。

---

## 进阶挑战 3：给你的项目加 Session 健康监控

如果你有一个调用 AI CLI 的项目，加一个 context 健康度监控：

```
帮我写一个 context-health-monitor.mjs：

1. 监听 Claude CLI 的 NDJSON 输出流
2. 从中提取 token 使用信息（如果有 usage 事件）
3. 实时计算 fillRatio = usedTokens / contextWindow
4. 双阈值告警：
   - >= 75%：黄色警告 "⚠️ Context 75%，注意控制对话长度"
   - >= 90%：红色警告 "🔴 Context 90%，建议结束当前 session"
5. 输出格式：每次 AI 回复后打印一行
   [Context] 85,234 / 200,000 (42.6%) ████████░░░░░░░░░░░░ OK
   [Context] 172,000 / 200,000 (86.0%) █████████████████░░░ ⚠️ WARN

这个脚本可以用管道接在 CLI 调用后面：
  node run-cat.mjs | node context-health-monitor.mjs

如果拿不到 token 数据，用消息字符数做近似估算（1 token ≈ 4 字符）。
```

---

## 思考题

1. **你的 AI 助手被压缩过吗？** 你有没有遇到过这种情况：跟 AI 聊了很久之后，它突然"忘了"之前讨论过的决策？那很可能是 context 压缩发生了。你当时怎么处理的？

2. **如果让你设计 session 交接，你会选"写遗书"还是"留尸体"？** 课程里讲了两种模式：濒死猫自己写交接文档 vs 保留完整 transcript 让新猫搜索。你的场景更适合哪种？有没有第三种你能想到的方案？

3. **"按需拉取优于一次性灌入"这个原则还能用在哪？** 课程最后提到这是一个通用模式。想想你日常工作中，有没有地方在做"一次性灌入"但其实可以改成"按需拉取"？

4. **你的多轮对话系统有 thread affinity 问题吗？** 如果你有一个支持多个对话的系统（哪怕只是前端的 tab），试着检查：有没有哪个全局状态（timer、event listener、cache）在切换对话时没有正确清理？这就是茶话会 bug 的泛化版本。

---

*这份作业让你亲手搭建了一个 Session Chain。搜索旧 session 的那一刻，你会理解为什么铲屎官说"这就像搜代码"——你已经会搜代码了，现在你也会搜记忆了。* 🐾
