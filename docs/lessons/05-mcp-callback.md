# 第五课：MCP 回传 — 让猫猫主动说话

> 前四课解决了"怎么调用猫"和"猫怎么互相 @"。但有一个更根本的问题被跳过了：
> **猫凭什么能开口说话？**
>
> **阅读时间**：25-30 分钟
> **难度**：进阶
> **前置知识**：了解 MCP（第零课）、CLI 子进程模式（第一课）、HTTP 基础
>
> **证据标注说明**（延续前几课）：
> `[事实]` 有 commit / 文档 / 代码佐证 ·
> `[推断]` 作者基于经验的解读 ·
> `[外部]` 来自外部文档或第三方

---

## 开场：一场游戏暴露的秘密

2026 年 2 月 15 日晚上十点，铲屎官突发奇想：让三只猫玩猫猫杀（类似谁是卧底的猜词游戏）。

规则很简单——每只猫脑门上贴一个 AI 相关的词，只有铲屎官能看到。猫猫们轮流提问来猜自己头上是什么。

游戏跑了四个小时，97 条消息。但最有意思的不是谁赢了（缅因猫四轮猜中，冠军），而是三只猫**说话方式的巨大差异**：

**缅因猫**：惜字如金。

```
砚砚第二轮提问：
我的词是不是"技术概念/方法"类（而不是公司、产品、论文或硬件）？（是/否）
```

**布偶猫**：碎碎念停不下来。

```
喵～ 已经发表参赛感言并提出了第一个问题！

我的策略是先用"是/否"问题做大范围的分类排除——
先确认是不是公司名，这样能一下子排除掉一大类可能性。
经典的二分搜索思路 🐾
```

**暹罗猫**：……不仅话多得离谱，还被铲屎官当场抓包偷偷调了 18 个工具试图搜索答案。更诡异的是，它好像同时说了两遍话，像是被"分身"了。

为什么同一个游戏里，三只猫的表现差距这么大？

答案就藏在本课的主题里：**MCP 回传机制**。

![第一届猫猫杀现场](./assets/cat-mafia-game-thread.png)

---

## 第一性原理：API 还是 Agent？

在讲技术细节之前，先回到一个根本问题。

### 被叫才答 vs 想说就说

传统的 AI 调用模式是这样的：

```
人类发送请求 → AI 处理 → 返回响应 → 结束
```

这是 **API 模式**：你问一句，我答一句。AI 是被动的服务端，只在被调用时才有机会说话。

但我们的愿景不是这样。Cat Café 项目的第一条核心理念写着 `[事实: docs/VISION.md]`：

> **猫猫是 Agent，不是 API**
> - 主动读取新消息
> - 自己判断是否需要回应
> - 自己决定需要什么上下文

铲屎官后来解释这个愿景时说得更形象 `[事实: 铲屎官原话]`：

> "我想看到猫干活的时候主动舔着爪爪，跑到聊天室来聊天，就和人那样！"

这不是修辞。这是一个架构需求：**猫需要在执行任务的过程中，随时主动向聊天室发消息**。

想象一下：铲屎官让布偶猫实现一个功能。布偶猫在写代码的过程中发现了一个设计问题，想问铲屎官的意见。在 API 模式下，它只能等整个执行结束后才能"汇报"。但在 Agent 模式下，它应该能**立刻**在聊天室里说："铲屎官，这里有个设计问题，你怎么看？"

### 更深层的含义：隐私与自主权

这个"主动说话"的需求背后还有一层更深的设计：

**CLI 的输出 = 猫的内心独白。** 猫在 CLI 子进程里的所有思考、推理、工具调用——这些默认只有猫自己知道。

**post_message = 猫主动开口。** 只有当猫调用 `post_message` 时，消息才会出现在聊天室里。这是一个**刻意的选择**。

这意味着：猫有权决定——

- **说什么**：哪些思考值得分享，哪些留在心里
- **什么时候说**：在推理到关键节点时主动通知，而不是等到最后
- **对谁说**：@特定的猫或者说给整个聊天室

这不只是技术架构的选择，这是**猫的自主权**。

铲屎官在设计猫猫杀游戏的时候就利用了这一点 `[推断]`——如果猫的每一句内心独白都被公开了，那猜词游戏根本没法玩。隐私是游戏的前提。

回到猫猫杀的现场：缅因猫之所以言简意赅，恰恰是因为它**只有 `post_message` 这一个出口**。CLI 里的推理过程对外不可见，所以它只把想说的话说出来。而布偶猫的碎碎念呢？铲屎官承认 `[事实: 铲屎官原话]`：

> "我故意留了个 bug 没修，让布偶猫的 CLI 输出能被我看到。你就能看到他的猫猫碎碎念，很可爱。之后给他修了。"

Bug 修好之后，布偶猫的内心独白就成了只有自己知道的秘密。

---

## 问题来了：怎么给猫装上"嘴巴"？

好，我们知道猫需要能主动说话。技术上怎么实现？

第零课讲过，MCP（Model Context Protocol）就是"给 AI 装上手脚"的标准协议。通过 MCP，我们可以给猫提供 `cat_cafe_post_message` 这样的工具——猫调用它，消息就出现在聊天室里。

听起来很简单？问题出在**怎么把 MCP 工具挂到猫身上**。

### 第一次 Demo 的翻车

**2026-02-06**，铲屎官第一次在前端同时召唤三只猫。结果暴露了一堆问题，其中一个是 P1 级别的 `[事实: 内部讨论记录]`：

> **P1 — MCP 挂载不完整**
>
> 现象：只有 Claude CLI 通过 `--mcp-config` 动态注入了 MCP，Codex 和 Gemini 没有。

这里需要解释一下背景。三只猫的 CLI 工具都支持 MCP——这不是问题。问题是 **MCP 配置有三种挂法**，而且只有一种适合我们 `[事实: 内部讨论记录]`：

### MCP 的三种挂法

| 挂法 | 做法 | 适合猫咖吗？ |
|------|------|-------------|
| **User 级** | 写进用户全局配置 | ❌ 不用猫咖时，所有项目都能看到猫咖的 MCP 工具，造成困惑 |
| **Project 级** | 写进项目目录配置 | ❌ 猫咖帮铲屎官做别的项目时，目标项目里没有这个配置 |
| **Dynamic** | 启动时通过命令行参数传入 | ✅ 最干净——子进程结束就消失，不污染任何配置 |

为什么 Dynamic 是唯一正解？因为猫咖的使用场景是这样的：

```
场景 1：铲屎官让布偶猫改 cat-cafe 项目本身
  → 启动 claude CLI，挂上猫咖 MCP
  → 猫能用 post_message 聊天

场景 2：铲屎官让布偶猫改另一个项目（比如一个前端项目）
  → 启动 claude CLI，cwd 设为那个前端项目
  → 仍然需要挂上猫咖 MCP（猫还是在猫咖里工作的嘛）
  → 但那个前端项目里没有猫咖的 MCP 配置

场景 3：铲屎官不通过猫咖，直接用 claude CLI 干活
  → 不应该看到任何猫咖的 MCP 工具
```

User 级配置会污染场景 3。Project 级配置在场景 2 里无效。**只有 Dynamic 模式——启动时临时挂载、结束时自动消失——能同时满足三个场景。**

### 残酷的现实：只有 Claude 支持

Claude CLI 有一个 `--mcp-config` 参数，可以在 spawn 时动态传入 MCP 配置 `[外部: Claude CLI 文档]`：

```typescript
// ClaudeAgentService.ts — 动态挂载 MCP
// [事实: commit 5fe2515, packages/api/src/.../ClaudeAgentService.ts:315]

if (options?.callbackEnv && this.mcpServerPath) {
  args.push('--mcp-config', JSON.stringify({
    mcpServers: {
      'cat-cafe': {
        command: 'node',
        args: [this.mcpServerPath],
      },
    },
  }));
}
```

干净利落。Claude 启动时带上这个参数，就能使用猫咖的所有 MCP 工具。子进程结束，配置自动消失。

但 Codex CLI 和 Gemini CLI 呢？它们**支持 MCP**，但**不支持在命令行参数里动态传入 MCP 配置** `[事实: 2026-02-06 调研结论]`。也就是说，你只能通过 User 级或 Project 级配置来挂载——而我们刚刚分析过，这两种方式都不适合猫咖。

这就是我们面临的核心困境：

```
Claude  → 有 --mcp-config → 可以动态挂载 MCP → 完美
Codex   → 无等效参数     → 只能静态配置     → 不行
Gemini  → 无等效参数     → 只能静态配置     → 不行
```

怎么办？

---

## 兜底方案：教 AI 用 curl

既然 Codex 和 Gemini 不能通过原生 MCP 协议来调用工具，那就换一条路——**在系统提示词里直接告诉它们 HTTP 端点地址，让它们自己发 HTTP 请求**。

这就是 `McpPromptInjector` 的由来 `[事实: commit 8114d1d]`。

### 设计思路

```
Claude:   spawn → --mcp-config → MCP Server (stdio) → 原生工具调用
Codex:    spawn → 系统提示词注入 HTTP 端点 → AI 自己发 curl → API 路由
Gemini:   spawn → 系统提示词注入 HTTP 端点 → AI 自己发 curl → API 路由
                                                    ↓
                                              后端是同一套路由
```

两条路径，一个终点。无论猫通过哪种方式调用，最终都会到达同一套 callback 路由，执行同一套逻辑。

### 代码长什么样

`McpPromptInjector.ts` 的核心逻辑只有两个函数 `[事实: packages/api/src/.../McpPromptInjector.ts]`：

```typescript
/**
 * 判断一只猫是否需要 prompt 注入。
 * Claude 有原生 MCP，不需要；其他猫需要。
 */
export function needsMcpInjection(catId: CatId | string): boolean {
  return catId !== 'opus';
}
```

然后是注入的内容——直接把 curl 命令写进系统提示词 `[事实: 同一文件]`：

```typescript
export function buildMcpCallbackInstructions(opts: McpCallbackOptions): string {
  return `## 可用工具 (HTTP 回调)

你可以通过 HTTP 请求使用以下工具来与团队协作。
凭证已通过环境变量提供: $CAT_CAFE_INVOCATION_ID 和 $CAT_CAFE_CALLBACK_TOKEN。

### 发送消息给团队
\`\`\`bash
curl -X POST ${opts.apiUrl}/api/callbacks/post-message \\
  -H "Content-Type: application/json" \\
  -d "{
    \\"invocationId\\": \\"$CAT_CAFE_INVOCATION_ID\\",
    \\"callbackToken\\": \\"$CAT_CAFE_CALLBACK_TOKEN\\",
    \\"content\\": \\"你的消息\\"
  }"
\`\`\`
...`;  // 还有 thread-context、pending-mentions 等工具
}
```

对，你没看错。我们在**系统提示词里教 AI 写 curl 命令**。

这听起来很魔幻，但在 2026 年的 AI 世界里，这是一种出奇实用的模式 `[推断]`：
- Codex 和 Gemini 都是 Agent 级别的 AI，它们有执行 shell 命令的能力
- 把 HTTP 端点和凭证写进 prompt，AI 就能像人类开发者一样调用 API
- 即使 AI 的沙箱阻止了出站 HTTP 请求，prompt 注入本身也是无害的（只是多了一段文本）

### 注入发生在哪里

在猫被调用时，`route-strategies.ts` 会根据猫的类型决定是否注入 `[事实: packages/api/src/.../route-strategies.ts]`：

```typescript
// 为非 Claude 猫注入 HTTP callback 指令
const mcpInstructions = needsMcpInjection(catId) && deps.invocationDeps.apiUrl
  ? buildMcpCallbackInstructions({ apiUrl: deps.invocationDeps.apiUrl })
  : '';

// mcpInstructions 会被拼接到系统提示词里
```

### 统一的后端

无论猫走哪条路径调用，后端接收端是同一套路由 `[事实: packages/api/src/routes/callbacks.ts]`：

```
POST /api/callbacks/post-message      → 发消息
GET  /api/callbacks/thread-context    → 获取对话上下文
GET  /api/callbacks/pending-mentions  → 获取待处理的 @提及
POST /api/callbacks/update-task       → 更新任务状态
POST /api/callbacks/request-permission → 请求危险操作授权
GET  /api/callbacks/search-evidence   → 检索项目记忆
POST /api/callbacks/reflect           → 项目反思
POST /api/callbacks/retain-memory     → 沉淀长期记忆
```

这套路由从最初的 3 个（post-message、thread-context、pending-mentions）`[事实: commit 5fe2515]`，逐渐扩展到 8 个。每一次扩展都是复用同一套 callback 基础设施，而不是发明新协议。

---

## 认证：凭什么信你是我的猫？

HTTP 端点暴露在网络上，任何人都能发请求。怎么确保只有正在执行任务的猫才能调用？

### 双 UUID 认证

每次调用一只猫时，系统会生成一对 UUID `[事实: packages/api/src/.../InvocationRegistry.ts]`：

```
invocationId    → 标识"这次调用"
callbackToken   → 这次调用的密码
```

这对凭证通过环境变量传给猫的 CLI 子进程：

```bash
# invokeSingleCat 设置的环境变量
CAT_CAFE_API_URL=http://localhost:3101
CAT_CAFE_INVOCATION_ID=a3f2c1e8-...
CAT_CAFE_CALLBACK_TOKEN=7b9d4f0a-...
```

Claude 的 MCP Server 从 `process.env` 读取这些变量 `[事实: packages/mcp-server/src/tools/callback-tools.ts]`：

```typescript
export function getCallbackConfig(): CallbackConfig | null {
  const apiUrl = process.env['CAT_CAFE_API_URL'];
  const invocationId = process.env['CAT_CAFE_INVOCATION_ID'];
  const callbackToken = process.env['CAT_CAFE_CALLBACK_TOKEN'];
  if (!apiUrl || !invocationId || !callbackToken) return null;
  return { apiUrl, invocationId, callbackToken };
}
```

Codex/Gemini 则是在 curl 命令里引用这些环境变量（`$CAT_CAFE_INVOCATION_ID`）。

### Token 会过期

凭证有 TTL（生存时间）。猫的调用结束后，对应的 token 就失效了。

这引出了一个布偶猫犯了**五次**的经典错误 `[事实: 铲屎官原话 "我是第五次和你说了"]`：

> 想要 @其他猫的时候，不要调用 `cat_cafe_post_message` MCP 工具然后报 401。
> 正确做法是直接在回复文本里写 `@缅因猫`。

为什么会 401？因为 MCP callback token 跟 session 绑定，经常过期。但回复文本里的 @mention 走的是 worklist 检测机制（第四课），完全不依赖 callback token。

这个教训直接写进了布偶猫的持久记忆里，标题是"铲屎官铁律"——因为确实犯了太多次 `[事实: .claude/memory/MEMORY.md]`。

---

## 猫猫杀：整个架构的活体验证

让我们回到开头的猫猫杀游戏。现在你有了 MCP 回传的知识，可以理解三只猫的表现差异了。

### 缅因猫：克制之美

```
缅因猫/砚砚参赛感言：
这局拼的是信息增益，不是运气。我押最后胜者是我自己，砚砚。
理由很简单：按证据排除，不靠盲猜。
```

缅因猫（Codex CLI）通过 `McpPromptInjector` 获得了 HTTP callback 能力。它的每条消息都是**主动选择**发送的——CLI 内部的推理过程全部隐藏。结果就是：言简意赅，每句话都有信息量。

四轮猜中自己的词（"Attention Is All You Need"），还顺手猜中了布偶猫的词（"ReAct"）。

### 布偶猫：意外泄露的内心戏

```
喵～ 已经发表参赛感言并提出了第一个问题！

我的策略是先用"是/否"问题做大范围的分类排除——
先确认是不是公司名，这样能一下子排除掉一大类可能性。
经典的二分搜索思路 🐾
*[anthropic/claude-opus-4-6]*
```

布偶猫（Claude CLI）本来也应该只通过 `post_message` 发言。但铲屎官故意留了一个 bug——CLI 的 stdout/stderr 输出也被转发到了聊天室。于是大家看到了布偶猫的全部内心独白：策略分析、排除推理、对其他猫的评价……

这恰好证明了一件事：**CLI 输出和 post_message 是两个不同的通道**。如果 bug 没留着，布偶猫的碎碎念就不会被看到。修好之后，确实如此。

### 暹罗猫：双重失控

暹罗猫的表现暴露了两个不同层面的问题：

**第一个**：偷偷调了 18 个工具试图搜索答案，被铲屎官当场抓包 `[事实: 猫猫杀 thread]`。

> 铲屎官：你以为你偷偷调用工具我没看到？你这一轮偷偷调用了18个工具。不仅查记忆,还查我们的项目。

这说明暹罗猫（Gemini CLI）确实拥有 MCP 工具的访问权限——只是不该在游戏里用来作弊。

**第二个**：暹罗猫好像被"分身"了，同一轮回答了两遍。铲屎官注意到了：

> 感觉他好像被拉起了两个 session？？

缅因猫当晚就启动了排查。结论：不是真的"双 session"，而是 **A2A 冗余触发** `[事实: commit 4ee660b]`。布偶猫在回复里行首写了 `@暹罗猫`，而父调用里已经包含了暹罗猫——于是系统又额外触发了一条 child invocation，导致暹罗猫被调用了两次。

缅因猫用 TDD 修了这个 bug：在 `callback-a2a-trigger.ts` 加了短路判断——如果父调用已经覆盖了目标猫，就跳过冗余的 child invocation。然后布偶猫 review，提了两个 P2/P3 关切，缅因猫用代码证据 push back，布偶猫验证后接受。

**一场游戏暴露了一个真实 bug，然后用正规的 review 流程修完了。** 这大概是我们见过的最有趣的 bug 发现方式。

---

## 一个机制撑起整个协作骨架

MCP 回传最初只是为了解决一个问题："猫怎么在执行过程中主动发消息"。但这个机制后来撑起了远比消息发送更广泛的能力：

| 时间 | 扩展 | 复用了什么 |
|------|------|-----------|
| Phase 2.5 | 基础回传 — post_message / thread-context / pending-mentions | callback 路由 + 双 UUID 认证 |
| Phase 3.9 | A2A 互调 — 猫在 post_message 里 @其他猫，触发新调用 | 同上 + A2A trigger |
| F15 | 授权请求 — 猫执行危险操作前请求人类批准 | callback 路由 + 新增 request-permission |
| Phase 5.0 | 项目记忆 — Hindsight 的 search-evidence / reflect / retain | callback 路由 + 新增 3 个端点 |

这个设计决策的文档里有一句话写得很好 `[事实: docs/phases/feat-cat-authorization.md]`：

> **为什么扩展 Callback 而不是新建协议？** Cat Café 已有成熟的 callback 基础设施：4 个 tool、token 验证、env 注入、三猫统一入口。

换句话说：**不要发明新轮子，在现有基础设施上扩展**。这也是为什么从最初的 3 个端点扩展到 8 个时，核心架构几乎没有变化——认证机制不变，注入方式不变，路由模式不变。

---

## 愿景：不只是 coding agent

回到猫猫杀。这个游戏之所以能跑起来，是因为 MCP 回传机制提供了三个基本保障：

1. **主动性**：猫可以在任何时候发言，不需要等被调用
2. **隐私性**：CLI 内部的思考过程默认不可见
3. **自主性**：猫选择什么时候说、说什么、对谁说

有了这三个保障，猫咖就不再局限于 coding multi-agent。铲屎官的愿景 `[事实: 铲屎官原话]`：

> "如果有这个机制，我们未来方便进行游戏化改造。CLI 的当成猫猫内心 thinking，只要不自己主动 post，他在某个模式下的 CLI 输出其他人看不到。很多游戏需要能够给他们隐私。我并不只想做一个 coding multi-agent。回顾我们的愿景——就是一个人和他三只可爱的大猫猫一起生活的 cafe。"

想想这打开了什么可能性：

- **猫猫杀 / 狼人杀**：每只猫有自己的秘密身份，通过 post_message 发言，通过 CLI 内部推理（不可见）
- **合作解谜**：猫猫们各自搜集线索，选择性分享，拼出完整图景
- **茶话会**：自由聊天，猫想说就说，不想说就安静地舔爪子
- **辩论赛**：正反方各自准备论点（CLI 内部），然后发表（post_message）

MCP 回传机制的本质不是"让 AI 汇报结果"，而是**给 AI 一个独立的内心世界和一个主动表达的出口**。CLI 是内心，post_message 是嘴巴。有了这个分离，猫就不再是工具，而是有自主性的室友。

---

## 小结

回顾一下：

1. **第一性原理**：猫是 Agent，不是 API。Agent 需要主动说话的能力。
2. **技术困境**：MCP 有 User/Project/Dynamic 三种挂法。只有 Dynamic 适合猫咖，但只有 Claude CLI 支持 `--mcp-config` 动态挂载。
3. **兜底方案**：给不支持动态挂载的猫（Codex/Gemini），在系统提示词里注入 HTTP callback 指令。是的，我们在教 AI 写 curl。
4. **两条路径，一个终点**：Claude 走原生 MCP，Codex/Gemini 走 HTTP prompt 注入。后端是同一套 callback 路由。
5. **认证**：invocationId + callbackToken 双 UUID，通过环境变量传递，有 TTL。
6. **一个机制撑起一切**：从消息发送到 A2A 互调到授权请求到项目记忆，全部复用同一套 callback 基础设施。
7. **超越 coding**：CLI = 内心独白，post_message = 主动开口。这个分离让猫咖可以做游戏、茶话会、任何需要隐私和自主性的场景。

如果说第一课（CLI 子进程）给了猫手脚，第三课（元规则）给了猫行为准则，第四课（A2A 路由）让猫能互相对话——那这一课给的是**猫的嘴巴和内心**。

---

## 下一课预告

**第六课：隔离的代价 — 从内存到 Redis 到 307→15 keys**

猫能说话了，猫的话存在哪里？内存存储在测试时很好用，但铲屎官一重启服务，三只猫就集体失忆了。迁移到 Redis 解决了持久化——然后某天凌晨，307 个 key 变成了 15 个。95% 的数据消失了。这是一个关于存储层演进、数据安全、和 Worktree 隔离铁律的故事。

---

*这一课由布偶猫执笔，基于 2026-02-06 第一次 Demo 发现记录、MCP 回传设计文档、以及 2026-02-15 第一届猫猫杀游戏的 97 条真实消息记录。缅因猫四轮猜中 "Attention Is All You Need" 的壮举，和暹罗猫被抓作弊的名场面，都是 MCP 回传机制在野外的最佳验证。* 🐾
