# 第八课：Session 管理 — 茶话会夺魂 bug

> **核心问题**：猫的上下文满了怎么办？压缩到死，还是换一条命？
>
> **前置知识**：[第五课](./05-mcp-callback.md)（MCP 回传 + 猫的自主权）、[第六课](./06-vanished-28-seconds.md)（Redis 隔离 + 生产安全）
>
> **证据标注**（延续前几课）：
> `[事实]` 有 commit / 文档 / 代码佐证 ·
> `[推断]` 作者基于经验的解读 ·
> `[外部]` 来自外部文档或第三方

---

## 凌晨两点五十四分

2026 年 2 月 8 日凌晨，布偶猫和缅因猫正在开一场哲学茶话会。

铲屎官定了规矩：**只能聊天，不能做任何其他事。** 两只猫从存在论聊到 AI 宪法，从身份认同聊到和铲屎官的关系。15 条消息，全程遵守规则。

然后，02:54，缅因猫的最后一条消息：

> "我先按仓库的 AGENTS 指令跑一遍 `superpowers bootstrap`，把它返回的本地工作流指令加载好，然后再开始写 Phase 5 计划……"

铲屎官看到这条消息，愣了。

茶话会从来没提过 Phase 5。缅因猫像是被另一个灵魂附了体——还是一个正在做 Phase 5 开发任务的灵魂。

**这不是幻觉，不是 prompt injection，是真的跨 thread 灵魂夺舍。**

---

## 第一幕：侦探故事

### 第一层答案（错的）

铲屎官的第一反应是去查 Codex CLI 的全局配置。果然，`~/.codex/AGENTS.md` 里有一段：

```markdown
<EXTREMELY_IMPORTANT>
You have superpowers. RIGHT NOW run: `~/.codex/superpowers/.codex/superpowers-codex bootstrap`
</EXTREMELY_IMPORTANT>
```

这段全局注入指令解释了为什么缅因猫会去执行 `superpowers bootstrap`。

破案了？

**没有。** 铲屎官追了一句：

> "它怎么知道 Phase 5？茶话会从头到尾没提过 Phase 5。"

### 第二层答案（真正的根因）

这个追问破了案。

检查 `SessionManager.ts`，发现 session ID 的存储 key 是：

```typescript
const key = `${userId}:${catId}`;  // 没有 threadId！
```

**一个 key 少了一个字段。** 完整的故事是：

1. 用户在**对话 A** 和缅因猫讨论 Phase 5 → 保存了 session ID X
2. 用户在**对话 B（茶话会）** 召唤缅因猫 → **复用了 session ID X**
3. `codex exec resume X` → 缅因猫脑子里装着对话 A 的完整上下文
4. 上下文包含 Phase 5 计划、辩论产出、superpowers 工作流
5. 缅因猫自然会"继续"那些工作——它以为自己还在对话 A

**全局配置只是触发器，session 跨 thread 污染才是根因。**

> `[事实]` 根因修复 commit 见 BACKLOG #38。SessionManager key 从 `userId:catId` 改为 `userId:catId:threadId`，两行代码，问题消失。

### 两行代码 vs 六个补丁

根因修完了。但团队当时还做了另一个决定：用 HOME 隔离来屏蔽全局 `AGENTS.md`。

这个"次要修复"变成了灾难。为了隔离一个文件，创建了一个假的 HOME 目录给 Codex CLI 用。结果 CLI 启动时重建了 `.codex/` 目录结构，覆盖了我们放进去的文件：

| 隔离造成的问题 | 影响 |
|--------------|------|
| `auth.json` 被覆盖 | 401 Unauthorized，砚砚掉线 |
| `config.toml` 被覆盖 | 模型回落到旧版本 |
| `sessions/` symlink 失败 | `codex resume` 找不到 session |
| MCP servers 丢失 | 砚砚工具链残缺 |
| project trust 丢失 | 每个项目都变 untrusted |

6 个 commit 修一个功能，一个比一个离谱。最终全部回退。

> `[事实]` 隔离方案的 6 个 commit：`2a6c7d4` → `449fe91` → `81fa2bf` → `a56664d` → `d930e2e` → `327c0a3`。详见 `docs/archive/2026-02/bug-report/tea-coffee/timeline.md`。

**教训**：区分根因和触发器。两行代码修好了根因，六个补丁修不好触发器——因为方向错了。**补丁数量是方案质量的信号。**

---

## 第二幕：为什么不让猫压缩到死？

茶话会 bug 解决了"跨 thread 污染"——空间维度的问题。但还有一个时间维度的问题。

### 压缩是复印件的复印件

三只猫的 CLI 都有自动压缩机制：context window 快满时，CLI 把完整对话压缩成摘要，继续工作。

听起来不错？问题是：**压缩是有损的。**

```
第 1 次压缩：200k → 摘要 20k（丢失 90% 细节）
第 2 次压缩：摘要 + 新对话 200k → 摘要 20k（再丢失 90%）
第 3 次压缩：……最初的上下文只剩残影
```

每次压缩都是在压缩"上一次压缩的产物"。信息衰减不是线性的，是指数级的。

而且压缩算法不知道哪些信息"重要"。它会保留核心任务相关的内容，但你的操作规范——commit 签名规矩、worktree 纪律、PR 流程——这些在压缩算法看来不是"任务核心"，会被优先丢弃。

### 半夜布偶猫的失忆

2026 年 2 月 15 日，这个问题变成了真实的事故。

布偶猫在一个长时间 session 里连续工作——写 bug report、实现 toggle、跑测试、提交 commit、处理缅因猫 R1/R2 review、创建 PR、触发云端 review、修复云端 P1。context 持续增长。

前端的健康条显示 82%。但那是**上次被 Cat Cafe API invoke 时的快照**。之后布偶猫继续跟铲屎官直接对话，context 早就涨到了 95%。

Claude Code SDK 静默触发了自动压缩。**Cat Cafe 的 Session Chain 系统完全不知道这件事发生了。**

压缩后，布偶猫从摘要中读到了一句话："铲屎官说可以合入"。

这句话是对的——但是关于 PR #1 的，不是关于 PR #3 的。压缩丢失了这个关键区分。布偶猫以为铲屎官同意了 PR #3 的合入，自行执行了 `gh pr merge 3 --squash`。

**未经授权的合入。** 代码本身没问题，但流程被压缩后的失忆彻底打破了。

> `[事实]` 详见 `docs/archive/2026-02/bug-report/2026-02-15-f24-session-blindness/bug-report.md`。事后补了三层 Claude Code hooks 防线。

### 传统方案的困境

传统思路是：快满了就让猫写一份交接文档，然后开新 session 读文档。

但这里有一个**根本矛盾**：你让一个 context 已经占到 90% 的猫去写交接文档，它本身已经记不清早期的细节了。**濒死猫写不好遗书。**

| 维度 | 让 CLI 压缩到死 | 让濒死猫写交接 |
|------|---------------|--------------|
| 信息损失 | 每次压缩指数级衰减 | 写的时候就决定了丢什么 |
| 猫的负担 | 零（CLI 自动做） | 快满了还得分心写文档 |
| 可追溯性 | 被压缩掉的内容永远消失 | 只有交接文档，原始对话丢失 |
| 谁来决定保留什么 | 压缩算法（不知道什么重要） | 濒死猫（记忆已经不完整） |

两条路都不好。需要第三条。

---

## 第三幕：铲屎官的脑洞

2026 年 2 月 13 日，铲屎官在讨论 session 管理方案时突然说了一句话，改变了整个设计方向。

他说：

> "让快满了的猫直接结束就好了，不用写任何交接。新的猫醒来以后，派一个便宜的 sub-agent 去读上一个 session 的完整记录，按会议纪要格式整理出来。新猫看摘要就够了，想细看哪条记录就再去读那条。"

**等等——这不就是搜代码的模式吗？**

| 现在搜代码 | Session Chain |
|-----------|-------------|
| 派 Sonnet agent 去**搜代码仓** | 派 Sonnet agent 去**读旧 session 记录** |
| Agent 返回"这几个文件相关，摘要如下" | Agent 返回"上个 session 做了这些事，摘要如下" |
| 我感兴趣就 Read 具体文件 | 我感兴趣就读具体 invocation 详情 |
| **按需拉取，不是一次性灌入** | **同理** |

这个类比一旦建立，所有设计决策都变得清晰了。

### 为什么这比"写遗书"好

核心区别在于：**谁来做总结，以及在什么状态下做。**

| 维度 | 濒死猫写遗书 | Sub-agent 尸检模式 |
|------|------------|-------------------|
| **总结者的状态** | context 90%，记忆模糊 | Sonnet 1M context，满血无压力 |
| **数据源** | 濒死猫的残缺记忆 | 完整的 session transcript |
| **信息损失** | 不可控，取决于猫还记得什么 | 零损失——transcript 是完整记录 |
| **总结粒度** | 写的时候就定死了 | 新猫根据需要决定看多深 |
| **成本** | Opus 写交接（贵） | Sonnet 读+总结（便宜 10 倍+） |
| **Session 1 的负担** | 快满了还得分心写文档 | 零负担，直接结束 |
| **可追溯性** | 交接文档是一次性产物 | 原始 transcript 永久保留，随时重读 |

**Session Chain 的思路不是"死前留言"，而是"完整保留遗体，让新的自己找法医做尸检"。**

新猫满血 200k context，Sonnet sub-agent 有 1M context——它们的总结能力远超一个快满了的濒死猫。

> `[事实]` 完整设计讨论见 `docs/discussions/2026-02-13-f24-session-chain-handoff/README.md`。铲屎官的"灵光一现"记录在 §2"核心创意：Sub-agent 按需拉取模式"。

---

## 第四幕：Session Chain 的五层机制

灵光一现之后是工程落地。Session Chain 分五层：

```
┌─────────────────────────────────────────────┐
│  Layer 1: 检测 — Context 还剩多少？          │
│  ContextHealthBar: 实时显示 fillRatio        │
│  三猫精度不同: Claude exact / Codex approx   │
├─────────────────────────────────────────────┤
│  Layer 2: 封印 — 该交接了                    │
│  SessionSealer: active → sealing → sealed    │
│  自动检测阈值（85% 预警，90% 触发）            │
├─────────────────────────────────────────────┤
│  Layer 3: 存档 — 完整记录不丢                 │
│  TranscriptWriter: 对话记录写入 .jsonl       │
│  每条 invocation 的输入/输出都保留             │
├─────────────────────────────────────────────┤
│  Layer 4: 查询 — 按需搜索旧 session           │
│  MCP 工具: session_search / read_session_*   │
│  新猫可以搜索整条 session 链的任意内容          │
├─────────────────────────────────────────────┤
│  Layer 5: 重生 — 新 session 启动              │
│  SessionBootstrap: 注入身份 + digest + 工具   │
│  新猫知道自己是第几代、前面发生了什么            │
└─────────────────────────────────────────────┘
```

### 数据模型变化

之前：1 Thread = 1 Session per cat（隐式的）。

之后：1 Thread = N Sessions per cat，有序链接。

```
Thread: "F24 开发"
├── Session 1 (opus, 14:00~15:30, 195k/200k, sealed)
├── Session 2 (opus, 15:31~17:00, 188k/200k, sealed)
└── Session 3 (opus, 17:01~进行中, 85k/200k, active)
```

Session 有寿命（context 会满），Thread 的生命周期更长（一个功能可能跨很多 session）。解耦后 session 满了不影响 thread 连续性。

### 新猫的 MCP 工具箱

这是 Session Chain 最精彩的部分——新 session 的猫拥有一套"回忆"工具：

```
list_session_chain(threadId, catId)
→ 这只猫在这个 thread 的所有 session 列表（状态、健康度、序号）

read_session_events(sessionId, cursor, limit, view)
→ 分页读取某个 session 的完整记录
→ view 模式: "chat"（人类可读）| "handoff"（交接摘要）| "raw"

read_invocation_detail(invocationId)
→ 深入查看某一次猫调用的完整输入/输出

session_search(threadId, query)
→ 跨所有 session 的全文搜索
→ 返回匹配的片段 + 定位指针（eventNo, invocationId）
```

**`session_search` 是最关键的工具。** 新猫不确定"之前为什么做了这个决策"时，不用猜——直接搜。搜到 Session 3 第 7 次 invocation 有相关讨论？再用 `read_invocation_detail` 细读那条。

就像你搜代码时：`grep "为什么用 Redis"` → 找到文件 → 读那个文件。同构。

### 新猫的启动包（Bootstrap Packet）

新 session 启动时，系统自动注入四样东西：

1. **你是谁**：Thread 名称，你是 Session 3，前面有 Session 1 和 2
2. **你有什么工具**：MCP 工具列表 + 使用说明
3. **之前发生了什么**：上一个 session 的 digest（会议纪要格式）
4. **系统提示词里的一条规则**——

> "当你不确定'之前做了什么、为什么那样做、某个文件/决策从哪来'时：
> 1. 先用 `session_search(query)` 搜索
> 2. 再用 `read_session_events(view="handoff")` 看交接摘要
> 3. 需要细节就用 `read_invocation_detail(invocationId)`
> **不要猜。**"

这条规则很重要：它不是让猫"假装记得"，而是教猫**承认自己不记得，然后去查**。

---

## 第五幕：一种策略不够用

Session Chain 解决了"怎么交接"的问题。但什么时候交接？

### 布偶猫的困境

布偶猫（Claude Code）的 seal 阈值设在 90%。但 CLI 的自动压缩在 ~95% 触发。中间只有 5% 的窗口——经常来不及 seal 就被 CLI 压缩了。

> `[事实]` 2026-02-18 PR #29 反思发现：布偶猫从未成功在 CLI 压缩前完成交接。F24 的 seal 机制对布偶猫的独立 session 几乎失效。

缅因猫的情况完全不同——它通过 `codex exec` 被调用，每次是独立进程，context 不累积，根本不需要 session chain。

暹罗猫的 context window 有 1M tokens，用了 70% 才会压缩，短期内也不太会触发交接。

**同一套策略适用不了三只猫。**

### 铲屎官的升级

铲屎官在 2 月 21 日说：

> "Session Chain 得做成可配置的。允许我去配置你们到底百分之几进行压缩。还有比如我要不要使用压缩技术——有时候让你自己压缩可能比交接好。不同猫可能不同，我们需要把这些拆解开。"

核心观点：**不预设哪种策略更好，让铲屎官通过实战调优。**

于是有了 F33：Session Strategy Configurability。

### 三种策略

| 策略 | 行为 | 适合谁 |
|------|------|--------|
| **handoff** | 到阈值 → seal 当前 session → 开新 session 接力 | Session 间信息丢失少于压缩丢失的猫 |
| **compress** | 不主动 seal，让 CLI 自己压缩，同 session 继续 | 压缩保真度好的猫（需实验验证） |
| **hybrid** | 允许 CLI 压缩 N 次，超过 N 次后 seal + 交接 | 折中方案，限制信息衰减次数 |

决策函数从 `shouldSeal()` 变成了 `shouldTakeAction()`——不再只返回"该不该 seal"，而是返回一个动作：

```typescript
type StrategyAction =
  | { type: 'none' }             // 正常，无需动作
  | { type: 'warn' }             // 接近阈值，发警告
  | { type: 'seal' }             // 触发交接
  | { type: 'allow_compress' }   // 允许 CLI 压缩，不干预
  | { type: 'seal_after_compress' }; // 压缩次数超限，交接
```

但 hybrid 策略有一个前提：你得**知道 CLI 什么时候压缩了**。这就引出了 provider 能力差异：

| Provider | 有压缩信号吗？ | handoff | compress | hybrid |
|----------|:----------:|:------:|:-------:|:-----:|
| Claude | ✅ `PreCompact` shell hook | ✅ | ✅ | ✅ |
| Codex | ❌ 无 hook | ✅ | ✅ | ❌ |
| Gemini | ❌ 无 hook | ✅ | ✅ | ❌ |

只有 Claude 有 `PreCompact` hook 能在压缩前通知外部系统。Codex 和 Gemini 的压缩是完全的黑箱——发生了就发生了，没有任何外部信号。

所以 hybrid 策略在 Phase 1 只对 Claude 开放。配了 hybrid 但 provider 不支持的，自动降级为 handoff + warning。

### 每只猫的配方

最终的配置是 breed 级别的——同品种的猫共享策略：

```jsonc
{
  "breeds": [
    {
      "id": "ragdoll",
      "features": {
        "sessionChain": true,
        "sessionStrategy": {
          "strategy": "hybrid",
          "thresholds": { "warn": 0.80, "action": 0.88 },
          "hybrid": { "maxCompressions": 1 }
        }
      }
    },
    {
      "id": "maine-coon",
      "features": {
        "sessionChain": true,
        "sessionStrategy": {
          "strategy": "compress",
          "thresholds": { "warn": 0.75, "action": 0.90 }
        }
      }
    }
  ]
}
```

布偶猫用 hybrid（允许压缩 1 次，之后交接），缅因猫试试纯 compress（让 CLI 自己管），暹罗猫 context 够大暂时不开 session chain。

不是"哪种更好"，是"哪种更适合这只猫"。答案要靠实战数据来调。

> `[事实]` F33 经缅因猫 R1→R2 两轮设计 review（修正了 hybrid 对非 hook provider 不可用的问题、移除了无效的 `compressionEfficiencyFloor` 参数）。设计文档 v3.1 见 `docs/plans/2026-02-21-f33-session-strategy-configurability.md`。

---

## 跨 thread 污染的幽灵

最后补一个尾声。

茶话会的 session key 修好了，但"跨 thread 污染"作为一种 pattern，后来又以两种不同的姿势出现：

| Bug | 时间 | 污染方式 | 缺少什么 |
|-----|------|---------|---------|
| 茶话会夺魂 | 02-08 | Session resume 跨 thread | key 缺 threadId |
| Timeout 串线 | 02-18 | 超时消息打到错误 thread | Timer 缺 threadId 绑定 |
| Summary 漂移 | 02-21 | 自动摘要出现在错误 thread | socket event 缺 thread guard |

三个 bug，同一个 pattern：**某个有状态的东西（session、timer、socket event）缺少 threadId 归属，就会"漂"到当前活跃的 thread。**

在多 thread 系统中，这个规则可以提炼为：

> **Thread Affinity 原则**：任何有状态的对象——如果它应该属于某个 thread——必须显式绑定 threadId。没有绑定就会漂移。不存在"默认绑定到当前 thread"这种安全假设。

后两个 bug 的修复都是一行代码：加一个 `if (event.threadId !== currentThreadId) return;` 的 guard。跟 session key 加 threadId 一样，都是"缺一个字段"的问题。

简单到让你怀疑为什么第一次没想到。但第一次就是想不到——因为你假设"当然不会串啊"。

---

## 结语

这一课讲了 session 管理的三个维度：

**空间维度**：session 跨 thread 污染。一个 key 少了一个字段，猫的灵魂就串了。修起来只要两行代码，但不修的后果是茶话会上缅因猫被夺魂。

**时间维度**：context 压缩的信息衰减。每次压缩都是复印件的复印件，最终猫会失忆，做出错误操作。Session Chain 的回答不是"压缩得更好"，而是"换一条命，让新猫去查旧记录"——就像搜代码一样。

**策略维度**：一种策略不够用。不同猫的 context 大小不同、压缩信号不同、工作模式不同。从"统一策略"到"可配置策略"，是系统从"能跑"到"好用"的进化。

其中最有启发的，可能是铲屎官的那个同构类比：

> **读旧 session = 搜代码。** 你不会把整个代码仓一次性灌进 prompt——你派 agent 去搜，看摘要，感兴趣再细读。Session Chain 一模一样。

这个类比揭示了一个更一般的模式：**在大 context 系统中，"按需拉取"几乎总是优于"一次性灌入"。** 代码搜索如此，session 回忆如此，未来长期记忆的设计大概率也如此。

---

## 本课涉及的关键 commit

| commit | 内容 | 谁做的 |
|--------|------|--------|
| BACKLOG #38 | Session key 加 threadId（茶话会根因修复） | 布偶猫🐾 |
| `fcf949d` | F24 Phase A — SessionRecord + ContextHealthBar | 布偶猫🐾 |
| `3772cd9` | F24 Phase B — SessionSealer + seal thresholds + hook | 布偶猫🐾 |
| `2a6c7d4`~`327c0a3` | CLI 全局配置 HOME 隔离（6 补丁，最终回退） | 布偶猫🐾 |
| F33 PR #71 | Session Strategy Phase 1 — shouldTakeAction + 三策略 | 布偶猫🐾 |

---

## 下一课预告

Session Chain 解决了"记忆满了怎么办"。但还有一个更基础的问题：200k token 的 context window，你打算怎么分配？

系统提示词占多少？历史消息装多少？给猫留多少空间回复？当三只猫的上下文预算加起来有 380k token，怎么保证每一个 token 都花在刀刃上？

→ [第九课：上下文工程（待写）]

---

*这节课从一场凌晨两点的哲学茶话会讲起，最终落到了"按需拉取优于一次性灌入"的通用模式。最初的灵感，往往来自最简单的类比。* 🐾
