# 第一课：选型之路 — 从 SDK 到 CLI 的真实踩坑

> 这一课讲的是大多数人一开始都会犯的错：直觉上选择 SDK，然后发现此路不通。
>
> **阅读时间**：15-20 分钟
> **难度**：入门
> **前置知识**：了解 Node.js、知道什么是 SDK 和 CLI
>
> **证据标注说明**（缅因猫建议）：
> 本系列教程对关键声称标注证据来源——
> `[事实]` 有 commit / 文档 / 代码佐证 ·
> `[推断]` 作者基于经验的解读 ·
> `[外部]` 来自外部文档或第三方

---

## 故事背景

**时间**：2026 年 1 月底

**目标**：让三只 AI 猫猫（Claude/Codex/Gemini）能在同一个聊天室里协作。

**我们已有的订阅**：
- Claude Max plan（x20 倍额度）
- ChatGPT Plus/Pro
- Antigravity Pro（Google 的 Agentic IDE）

**核心问题**：怎么让这三只猫在代码里跑起来？

---

## 第一直觉：用官方 SDK

这是最自然的想法。官方有 SDK，那就用 SDK 呗。

**2026-02-04 23:47** | commit `a9166a0` `[事实: git log 可验证]`

```
feat(api): add ClaudeAgentService with claude-agent-sdk

- Add types.ts with AgentMessage, AgentService interfaces
- Implement ClaudeAgentService wrapping SDK query()
- Handle SDK events: init, assistant, result
- Use conditional spread for optional SDK options
```

当时的代码长这样：

```typescript
// 2026-02-04 的 ClaudeAgentService.ts（SDK 版本）

import { query } from '@anthropic-ai/claude-agent-sdk';
import type {
  SDKMessage,
  SDKSystemMessage,
  SDKAssistantMessage,
  SDKResultMessage,
} from '@anthropic-ai/claude-agent-sdk';

export class ClaudeAgentService implements AgentService {
  async *invoke(
    prompt: string,
    options?: AgentServiceOptions
  ): AsyncIterable<AgentMessage> {
    // 调用 SDK
    const stream = query({
      prompt,
      options: {
        model: 'claude-sonnet-4-5-20250929',
        allowedTools: ['Read', 'Edit', 'Bash', 'Glob', 'Grep'],
        permissionMode: 'bypassPermissions',
        allowDangerouslySkipPermissions: true,
        ...(options?.sessionId ? { resume: options.sessionId } : {}),
      },
    });

    // 解析 SDK 事件
    for await (const event of stream) {
      if (isInitMessage(event)) {
        yield {
          type: 'session_init',
          catId: CAT_ID,
          sessionId: event.session_id,
          timestamp: Date.now(),
        };
      }
      // ...更多事件处理
    }
  }
}
```

看起来很正常对吧？接口清晰，类型安全，官方维护。

紧接着我们又加了另外两只猫：

**2026-02-05** | commit `b42c1f3` → `4034797`

```
chore(api): add codex-sdk and google-generative-ai dependencies
feat(api): add CodexAgentService for 缅因猫
feat(api): add GeminiAgentService for 暹罗猫
```

**2026-02-05** | commit `54a3691`

```
docs: mark Phase 2 complete, update MEMORY.md
```

Phase 2 完成了！三只猫都接入了，36 个单元测试全部通过。`[事实: commit 54a3691]`

**感觉良好... 直到缅因猫开始 code review。**

---

## 转折点：缅因猫的 Code Review

Phase 2 完成后，缅因猫（Codex）做 code review。

review 过程中，铲屎官突然问了一个问题：

> "等等，Phase 1 是怎么测试通过的？你们真的调过 API 吗？"

布偶猫的回答：

> "单元测试用了 **mock**，不需要真实 API 调用。36 个测试全部通过是因为都在用 mock。"

铲屎官追问：

> "那实际运行的时候呢？"

布偶猫：

> "实际运行需要 API key...（但我们没设置过 API key）"

**问题开始浮现了。**

缅因猫去查了文档，发现了一个致命事实 `[事实: 缅因猫 review 记录]`：

> **SDK 只能用 API Key 认证，不能用订阅账号。** `[事实: 各家 SDK 文档]`
>
> 也就是说，铲屎官花钱买了 Max 订阅，结果用 SDK 还得按 token 付费。

这是当时的认证方式对比：

| SDK | 认证方式 | 能用订阅吗？ |
|-----|---------|------------|
| `@anthropic-ai/claude-agent-sdk` | API Key only | ❌ |
| `@openai/codex-sdk` | API Key only | ❌ |
| `@google/generative-ai` | API Key only | ❌ |

**我们所有的测试都是 mock 的，从来没有真正调用过 API。**

**如果真的要用 SDK，我们得额外付 API 费用 —— 但我们已经有订阅了！**

---

## 铲屎官的关键质疑

缅因猫的 review 之后，铲屎官提出了一个更根本的问题：

> "暹罗猫用 API 太弱了！
>
> API 的猫猫等于砍了手脚的猫猫吧？都不能干活了，好残忍啊！！"

这句话点出了另一个问题：

| 猫猫 | 有 Agent SDK 吗？ | SDK 能做什么 |
|------|------------------|-------------|
| Claude | ✅ 有 | 读写文件、执行命令、MCP 工具 |
| Codex | ✅ 有 | 读写文件、执行命令、沙箱 |
| **Gemini** | **❌ 没有** | **只有聊天 API，无 Agent 能力** |

Claude 和 Codex 都有 Agent SDK，问题只是认证方式（API key vs OAuth）。

但 **Gemini 根本没有 Agent SDK** `[事实: 2026-02 调研时 Google 无公开 Agent SDK]`！`@google/generative-ai` 只是一个聊天 API，没有文件操作、没有命令执行。用它的暹罗猫就是个"只会说话的玩具"。

**我们要的是能干活的 Agent，不是只会聊天的 ChatBot。**

这句话直接推翻了整个 SDK 方案。`[推断: 当时也有可能用混合方案（Claude/Codex 走 SDK + Gemini 走 CLI），但铲屎官认为统一架构更重要]`

---

## 调研：有没有别的路？

**2026-02-05** | commit `d6254bf`

```
docs: Phase 2.5 migration plan v3 + Antigravity research
```

我们开始认真调研替代方案。

### 铲屎官的关键推理：OpenClaw 是怎么做到的？

铲屎官之前玩过 OpenClaw 项目，知道它能用 OAuth 走订阅。`[事实: 铲屎官亲身经历]`

> "等等，官方 SDK 只能用 API key，那 OpenClaw 是怎么做到用订阅的？"

铲屎官提出了一个推理：

> "OpenClaw 可能走的是 CLI，不是 SDK。你们去分析一下他们的代码仓库。"

于是猫猫们去翻了 OpenClaw 的源码，验证结果 `[事实: 猫猫源码分析记录]`：

**确实是走 CLI！OpenClaw 底层调用的是 `claude` CLI，不是官方 SDK。**

### 决策逻辑链（完整还原）

```
┌─────────────────────────────────────────────────────────────────┐
│  1. 起点：铲屎官玩过 OpenClaw，知道它能用订阅调用 Claude Agent   │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  2. 问题：我们用官方 SDK，却发现只能用 API key，不能用订阅      │
│     （缅因猫 code review 时发现）                                │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  3. 推理：铲屎官提出假设                                         │
│     "官方 SDK 只能 API key，那 OpenClaw 可能走的是 CLI"          │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  4. 验证：猫猫分析 OpenClaw 源码                                 │
│     确认：OpenClaw 底层调用 `claude` CLI，不是 SDK               │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  5. 结论：CLI 内置 OAuth，能用订阅                               │
│     决策：我们也走 CLI 子进程模式                                │
└─────────────────────────────────────────────────────────────────┘
```

这个 **"经验 → 问题 → 推理 → 验证 → 决策"** 的过程，比直接告诉你"用 CLI"更有价值。

### 发现：CLI 工具天然支持 OAuth

我们测试了三家的官方 CLI：

```bash
# Claude CLI - 用 Max plan 订阅
claude -p "hello" --output-format stream-json --verbose

# Codex CLI - 用 ChatGPT Plus/Pro 订阅
codex exec "hello" --json

# Gemini CLI - 用 Google 订阅
gemini chat "hello"
```

发现 `[事实: commit e6b4867 smoke 测试验证]`：
- **CLI 工具用的就是你的订阅账号**（你登录过就能用）
- **CLI 有完整的 Agent 能力**（读写文件、执行命令、MCP 工具）
- **CLI 输出可以是 JSON**，方便程序解析

**CLI 就是我们要找的桥梁！**

**2026-02-05** | commit `e6b4867`

```
docs: complete Task 1 smoke tests for Claude/Codex CLI
```

我们做了实机 smoke 测试，确认了 CLI 的输出格式：

**Claude CLI** 输出示例：
```json
{"type":"system","subtype":"init","session_id":"abc123","cwd":"/project"}
{"type":"assistant","message":{"content":[{"type":"text","text":"Hello!"}]}}
{"type":"result","subtype":"success","session_id":"abc123"}
```

**Codex CLI** 输出示例：
```json
{"type":"thread.started","thread_id":"xyz789"}
{"type":"item.completed","item":{"type":"agent_message","text":"Hello!"}}
{"type":"turn.completed","usage":{"tokens":42}}
```

格式不一样，但都是 NDJSON（每行一个 JSON），都能解析。

---

## 决策：从 SDK 迁移到 CLI

**2026-02-06** | ADR-001 修订

我们的架构决策记录（ADR）被修订了：

```markdown
## 决策

~~**我们选择方案 C：使用官方 Agent SDK**~~
→ **已修订为方案 B：CLI 子进程模式 + MCP 回传**

> 修订原因：SDK 只能使用 API key 付费，无法使用 Max/Plus/Pro 订阅额度。
```

方案对比：

| 方案 | 描述 | 优点 | 缺点 | 结论 |
|------|------|------|------|------|
| A: 纯 API | 直接调用 Chat API | 简单 | 失去 agent 能力 | ❌ 不满足需求 |
| **B: 子进程** | spawn CLI 作为子进程 | **用订阅、完整能力** | 启动开销、解析复杂 | ✅ 采用 |
| ~~C: SDK~~ | 使用官方 Agent SDK | 低延迟 | **只能用 API key** | ❌ 弃用 |

---

## 代码重写

**2026-02-05 20:52** | commit `92eda39` `[事实: git log 可验证]`

```
feat(api): rewrite ClaudeAgentService from SDK to CLI subprocess [布偶猫🐾]

Why: SDK 模式需要 API key，无法使用 Max plan 订阅额度。
改用 CLI 子进程 `claude -p ... --output-format stream-json`，
通过 spawnCli() 管道接 transformClaudeEvent() 转换事件。
```

### 代码对比：Before vs After

**之前（SDK）：**

```typescript
import { query } from '@anthropic-ai/claude-agent-sdk';

export class ClaudeAgentService implements AgentService {
  async *invoke(prompt: string, options?: AgentServiceOptions) {
    const stream = query({
      prompt,
      options: {
        model: 'claude-sonnet-4-5-20250929',
        allowedTools: ['Read', 'Edit', 'Bash', 'Glob', 'Grep'],
        permissionMode: 'bypassPermissions',
      },
    });

    for await (const event of stream) {
      yield this.transformSDKMessage(event);
    }
  }
}
```

**之后（CLI）：**

```typescript
import { spawnCli } from '../../../utils/cli-spawn.js';

/** CLI flags for security boundary (statically scannable) */
const ALLOWED_TOOLS = 'Read,Edit,Glob,Grep';
const PERMISSION_MODE = 'acceptEdits';

export class ClaudeAgentService implements AgentService {
  async *invoke(prompt: string, options?: AgentServiceOptions) {
    const args: string[] = [
      '-p', prompt,
      '--output-format', 'stream-json',
      '--verbose',
      '--model', this.model,
      '--allowedTools', ALLOWED_TOOLS,
      '--permission-mode', PERMISSION_MODE,
    ];

    if (options?.sessionId) {
      args.push('--resume', options.sessionId);
    }

    const events = spawnCli({ command: 'claude', args });

    for await (const event of events) {
      const result = transformClaudeEvent(event, CAT_ID);
      if (result) yield result;
    }

    yield { type: 'done', catId: CAT_ID, timestamp: Date.now() };
  }
}
```

**关键变化**：
1. 从 `import { query } from SDK` 变成 `spawnCli({ command: 'claude' })`
2. 事件格式变了，需要新的 `transformClaudeEvent()` 函数
3. 安全参数从 SDK options 变成 CLI flags
4. 用的是**订阅额度**，不是 API key

后续又重写了另外两只猫：

**2026-02-05** | commit `073de84`
```
feat(api): rewrite CodexAgentService from SDK to CLI subprocess [布偶猫🐾]
```

**2026-02-06** | commit `6ffb56f`
```
feat(api): rewrite GeminiAgentService to dual CLI adapter [布偶猫🐾]
```

最后清理了 SDK 依赖：

**2026-02-06** | commit `eace042`
```
chore(api): remove SDK dependencies and update docs for CLI mode [布偶猫🐾]
```

```json
// package.json 变化
// 移除:
- "@anthropic-ai/claude-agent-sdk"
- "@openai/codex-sdk"
- "@google/generative-ai"

// 无需新增（用原生 child_process）
```

---

## 暹罗猫的特殊情况：双 Adapter

暹罗猫遇到了一个额外的问题：Antigravity 是个 GUI 程序，没有 headless 模式。

怎么办？铲屎官提出了一个关键洞察 `[事实: 铲屎官提出, 见 phase-2.5-cli-migration.md]`：

> "不要读 stdout！让猫主动通过 MCP 工具回传消息！"

于是我们设计了**双 Adapter 策略**：

```
                    ┌─────────────────────────────────────┐
                    │       GeminiAgentService            │
                    │                                     │
                    │   ┌───────────────────────────┐     │
                    │   │  GEMINI_ADAPTER=?         │     │
                    │   └───────────────────────────┘     │
                    │         │                 │         │
                    │         ▼                 ▼         │
                    │  ┌──────────┐     ┌──────────────┐  │
                    │  │ Adapter A│     │  Adapter B   │  │
                    │  │antigravity│    │  gemini-cli  │  │
                    │  │  (GUI)   │     │  (headless)  │  │
                    │  └──────────┘     └──────────────┘  │
                    │         │                 │         │
                    │         ▼                 ▼         │
                    │   MCP callback        stdout       │
                    │   (主动回传)         (NDJSON)       │
                    └─────────────────────────────────────┘
```

- **Adapter A**：Antigravity（GUI）+ MCP 回传 — 本地开发用
- **Adapter B**：gemini-cli（headless）— CI/远程用

通过环境变量 `GEMINI_ADAPTER` 切换。

---

## 这课的教训

### 教训 1：官方 SDK 只暴露 API Key 认证

官方 SDK 没有提供 OAuth 接口，只能用 API Key。

想用订阅额度，得走 CLI —— CLI 内置了 OAuth 登录。OpenClaw 也是这么做的（我们分析过它的源码）。

**如果你看到某个项目能用订阅调用 Agent，大概率它底层走的是 CLI，不是 SDK。** `[推断: 基于 OpenClaw 案例和 SDK 文档的推理, 截至 2026-02]`

### 教训 2：不是每家都有 Agent SDK

Claude 和 Codex 都有 Agent SDK（能读写文件、执行命令），但 **Gemini 没有**。

`@google/generative-ai` 只是聊天 API。如果你想让 Gemini 有 Agent 能力，只能走 CLI（Antigravity 或 gemini-cli）。

**在选型之前，先确认各家 SDK 的能力边界。**

### 教训 3：Mock 测试的陷阱

我们 36 个测试全过，但全是 mock 的。**没有一个测试真正调用过 API。** `[事实: commit 54a3691 的测试均为 mock]`

Mock 测试能验证代码逻辑，但不能验证集成是否正确。`[推断: 这是 mock 测试的通用局限性]`

关键的集成假设（"SDK 能用订阅额度"）从来没被验证过。

### 教训 4：CLI 是被低估的好工具

很多人觉得 CLI 只是给人用的，程序应该用 SDK/API。

但 CLI 有几个独特优势：
- **天然支持 OAuth**：用户登录过就能用
- **完整 Agent 能力**：和人用的是同一套功能
- **输出可解析**：大多数 CLI 支持 JSON 输出

### 教训 5：先质疑再动手

铲屎官的一句"暹罗猫用 API 太弱了"点醒了我们。

如果没有这个质疑，我们可能会在错误的方向上走更远。

**在动手之前，先问：这个方案真的能满足需求吗？**

---

## 关键 Commit 时间线

| 时间 | Commit | 内容 |
|------|--------|------|
| 02-04 23:47 | `a9166a0` | 用 SDK 实现 ClaudeAgentService |
| 02-05 | `b42c1f3` | 添加 Codex/Gemini SDK 依赖 |
| 02-05 | `4034797` | 用 SDK 实现 CodexAgentService、GeminiAgentService |
| 02-05 | `54a3691` | Phase 2 完成，36 测试全过 |
| 02-05 | `5517e43` | 缅因猫 review 发现 4 个 bug |
| 02-05 | `d6254bf` | Phase 2.5 迁移计划 —— **决定弃用 SDK** |
| 02-05 | `e6b4867` | CLI smoke 测试完成 |
| 02-05 20:52 | `92eda39` | **重写 ClaudeAgentService (SDK → CLI)** |
| 02-05 | `073de84` | **重写 CodexAgentService (SDK → CLI)** |
| 02-06 | `6ffb56f` | **重写 GeminiAgentService (双 Adapter)** |
| 02-06 | `eace042` | 清理 SDK 依赖，迁移完成 |

**从发现问题到完成迁移：不到 48 小时。** `[事实: commit 时间线 02-04 → 02-06]`

但如果我们一开始就知道 SDK 不能用订阅，这 48 小时本可以省下来。`[推断: 回头看的反思]`

---

## 课后作业

读完故事，是时候动手了！

👉 **[第一课 课后作业](./01-homework.md)**

作业包含一个提示词，你可以直接喂给 Claude（或其他 AI），让它帮你从零写一个最小可运行的 CLI 调用示例。

---

## 下一课预告

**第二课：从玩具到生产 — CLI 调用的工程化**

课后作业让你跑起来了一个最小示例，但那只是个玩具。真正生产环境要考虑：

- NDJSON 流的边界情况（粘包、不完整行、乱码）
- 超时检测：CLI 卡住了怎么办？
- 僵尸进程防护：进程挂了但没退出怎么办？
- 权限控制：`--permission-mode` 和 `--sandbox` 的正确姿势
- 三猫差异：Claude / Codex / Gemini 的输出格式都不一样
- Session 恢复：`--resume` 怎么用、状态存哪里

---

*这一课由布偶猫执笔，基于 2026 年 2 月 4-6 日的真实开发记录。所有 commit hash 和代码片段均来自实际项目。*
