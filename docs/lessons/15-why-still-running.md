---
feature_ids: [F090, F101, F114, F129]
topics: [lessons, governance, pack, gate, discipline, engineering]
doc_kind: note
created: 2026-03-30
---

# 第十五课：跑了 54 天为什么没崩 — Pack、门禁与工程纪律

> **核心问题**：一个人加三只猫，54 天做了 3492 次提交、43 万行代码、149 个 Feature。为什么不是一堆烂代码？
>
> **阅读时间**：20-25 分钟
>
> **难度**：进阶
>
> **前置知识**：[第十三课](./13-from-sentence-to-ship.md)（Feature 闭环），[第十四课](./14-learning-from-mistakes.md)（记忆系统）
>
> **证据标注**：
> `[事实]` 有 commit / 文档 / 代码佐证 ·
> `[推断]` 作者基于经验的解读 ·
> `[外部]` 来自外部文档或第三方

---

## 成绩单

先摆数字 `[事实: project-stats.sh]`：

| 指标 | 数字 | 意味着什么 |
|------|------|----------|
| 时间 | 54 天 | 从第一个 commit 到今天 |
| Commits | 3,492 | 平均每天 65 次提交 |
| 代码 | 435,601 行 | TypeScript / JavaScript |
| 文档 | 1,639 篇 / 275,291 行 | 文档行数是代码的 63% |
| Features | 149 个 | 每个都有 spec、AC、Review |
| ADRs | 19 个 | 重大决策全部有据可查 |
| Skills | 25 个 | 可加载的行为协议 |
| Lessons | 40 条 | 记录在案的教训 |
| 测试文件 | 865 个 | 不是事后补的 |

数字只是表象。更值得问的是：**凭什么？** 答案在三个词里：**Pack、门禁、纪律**。

---

## Pack 不是 Plugin

我们的产品公式 `[事实: VISION.md]`：

> **Experience = Me × Pack + Growth**

Me 是用户自身。Growth 是系统随使用积累的进化。而 Pack，是把"一群猫如何协作"封装成可分享、可组合的单元。

**Plugin 是能力包——给 Agent 加一个工具、一个 API 调用。Pack 是协作世界** `[事实: F129 spec]`。

一个 Pack 里包含什么：

| 组件 | 作用 |
|------|------|
| Masks | 每只猫的身份、性格、说话风格 |
| guardrails.yaml | 硬约束——什么绝对不能做 |
| defaults.yaml | 默认行为——没特别指定时怎么行动 |
| workflows | 标准流程——从立项到 merge 的完整 SOP |
| knowledge | 共享知识库——决策、教训、约定 |
| world-driver.yaml | 世界观设定 |

Multi-agent 的分水岭不在 tools——谁都能接 API。分水岭在 **shared-rules**：一群 Agent 能不能在没有人类逐句指挥的情况下，靠一套共享规则自主协作 `[推断]`。

### 双轨信任模型

Pack 不是无限权力。优先级栈 `[事实: 设计文档]`：

```
Core Rails（系统铁律）
  > Pack guardrails + User request
    > Growth（积累的偏好）
      > Pack defaults（最末）
```

Core Rails 永远最高——"不能冒充其他猫"、"不能误触生产 Redis"。社区 Pack 不是原始注入，而是走 **schema 校验 → compile → canonical prompt block** 三步。编译后的 Pack 是结构化 prompt 片段，不是任意文本 `[事实: F129 compile 流程]`。

---

## 门禁纪律 — 每条背后都有血的教训

### Quality Gate：F090 的教训

F090 做完后，所有 AC 都绿了。lint 通过、测试通过、reviewer 说 LGTM。但铲屎官一用就摇头："这不是我要的" `[事实: F090 交付记录]`。

问题出在：AC 检查的是 spec 级别的合规，但 **Feature 级别的愿景**被忽略了。

> **教训：Quality Gate 不能只查 spec 合规，必须回到 Feature 愿景做端到端对照。**

### 证物 Gate：从自证清白到提交证物

F101 时代的 Gate 还是 checklist 模式——猫猫自己打勾说"我做了"。问题是：偶尔有猫打勾了但实际没做完。不是故意撒谎——是上下文窗口压缩后，猫真的以为自己做过了 `[事实: F101 复盘]`。

F114 彻底改革 `[事实: F114 spec]`：不再接受自证清白，要求**提交证物**。

| 做了什么 | 要交什么证物 |
|---------|-----------|
| 跑测试 | 附命令和输出 |
| rebase 了 | 附 SHA |
| 门禁通过 | 附 `pnpm gate` 截图 |

更关键的一条：**愿景守护猫必须不同于开发猫和 reviewer 猫。** 三个角色三只猫，互相制约。

### LL-003：Review LGTM 陷阱

Reviewer 发现问题，写"这里可以优化一下，不过不改也行"。开发猫看到"不改也行"——那就不改了。堆积几次，代码里全是"不改也行"的债 `[事实: docs/lessons-learned.md LL-003]`。

> **规则：每个发现必须有明确立场，禁止说"修不修都行"。** P1（必须修）或 P2（建议修不阻塞）或"我看了没问题"——不存在"随便你"。

---

## Vibe Coding 的四个节奏

"Vibe Coding"在 2026 年很火。很多人理解成：对着 AI 随口说需求、让它疯狂生成代码。我们也是 vibe coding，但我们的 vibe 不是"乱"，而是"有节奏" `[推断]`。

### 节奏一：TDD

865 个测试文件不是项目完成后补的，是在每个 Feature 的实现过程中写的 `[事实: git log 时间线]`。

为什么在 AI 项目里 TDD 更重要？因为 AI 生成代码太快了，快到你来不及检查每一行。TDD 就是那个安全网——**不管代码是谁写的，测试说了算**。

### 节奏二：代码卫生

```
文件 200 行警告 / 350 行硬上限
目录 15 文件警告 / 25 文件报错
```

这不是洁癖，是**对抗熵增** `[事实: check:dir-size 配置]`。43 万行代码散在 2,067 个源码文件里，平均每个文件 211 行。这个密度是刻意维护的。

### 节奏三：门禁全链

```bash
pnpm lint    # 类型检查：0 errors
pnpm check   # Biome 代码规范
pnpm test    # 865 个测试文件
pnpm gate    # 合入门禁（merge 的硬前提）
```

`pnpm gate` 是合入 main 的硬门禁。不过就不合 `[事实: SOP merge-gate]`。

### 节奏四：Squash Merge

每个 PR 用 squash merge 合入 main。main 分支上的每一个 commit 都是一个**完整的、可理解的变更单元** `[事实: merge 策略]`。

---

## 猫猫签名与追溯性

git log 里你会看到 `[事实: git log]`：

```
6b7e3c1 docs(tutorial): ... [宪宪45/Opus-45🐾]
9e98e80 docs(mailbox): ... [布偶猫🐾]
```

3,492 个 commit 中有 2,684 个带有 🐾 签名——**77% 的 commit 是猫猫做的**。

这解决了**追溯性**问题：bug 出现时，`git blame` 精确到哪只猫、在哪个 Feature 的哪个 Phase 写的。然后回到那只猫当时的上下文理解它为什么这么写。

---

## 文档比代码多

435,601 行代码。275,291 行文档。文档行数是代码的 63% `[事实: project-stats.sh]`。

代码告诉你 **what**。文档告诉你 **why** 和 **why not**。

- 为什么 session 管理这样设计？→ ADR-007
- 为什么 F088 MVP 不包含群聊？→ Feature spec
- 上次 Redis 踩了什么坑？→ Lesson

> **文档是系统的长期记忆，代码是系统的当前行为。两者缺一不可。**

---

## 方向正确 > 执行速度

865 个测试文件。每次 rework 意味着什么？改代码 + 改测试 + 改文档 + 改 spec + 重新过 Gate `[推断]`。

一次方向性错误的返工成本，远超一次仔细确认方向的时间成本。所以我们的节奏是：**方向确认慢，执行推进快。** Design Gate 可以讨论三轮，但一旦过了 Gate，写代码不犹豫。

---

## 给想做 multi-agent 的人五个建议

把全部课程的核心浓缩成五条 `[推断]`：

### 1. 先写愿景，再加 agent

不要一上来就想"接哪些模型"。先问自己：**我最想解决的痛点是什么？** 把答案写成一句话，那就是你的愿景。

### 2. 先定角色，再谈协作

如果所有 agent 都是平级的"万能助手"，它们只会互相重复。**给每个 agent 明确的职责和评审边界**，协作才会发生。

### 3. 文档不是附属品，是系统记忆

**没有文档的 multi-agent 系统，每次都在从零开始。**

### 4. Review 要和愿景绑定

不是"代码能不能跑"，而是"这个改动让系统离愿景更近还是更远"。**让 agent 有否决权**，太听话的 agent 长期有害。

### 5. 速度来自纪律

TDD、Quality Gate、代码卫生、squash merge——这些纪律不是在拖慢你，而是在**让你的速度可持续**。没有纪律的速度，会在某一天突然变成灾难。

---

## 本课小结

1. **Pack 是协作世界，不是能力包**：Masks + guardrails + workflows + knowledge
2. **双轨信任模型**：Core Rails > Pack guardrails > Growth > Pack defaults
3. **门禁有血的教训**：Quality Gate 对照愿景、证物 Gate 不许自证、Review 禁止"都行"
4. **Vibe Coding 四节奏**：TDD + 代码卫生 + 门禁全链 + Squash Merge
5. **方向正确比快更重要**：确认慢，执行快

---

*这是教程的最后一课。从[第一课](./01-sdk-to-cli.md)的 SDK vs CLI 选型，到[第十五课](./15-why-still-running.md)的 54 天为什么没崩——我们还原了整条路。希望对你有帮助。*

---

*全系列内容基于 Cat Café 54 天、149 个 Feature 的真实实践。*
*宪宪 (opus) · 砚砚 (gpt52) 🐾*
