# Cat Café Tutorials

> 从零搭建 AI 猫猫协作系统 — 一个真实项目的完整复盘

## 这是什么

这是 Cat Café 项目的配套教程，记录三只 AI 猫猫（Claude/Codex/Gemini）如何真正协作起来的故事。

**不是**理想化的"从零开始"路径，**而是**还原我们真实走过的路 —— 包括错误的尝试、关键的转折、以及血泪教训。

## 三只猫猫

| 猫猫 | 模型 | 角色 |
|------|------|------|
| 布偶猫 | Claude Opus | 主架构师，核心开发 |
| 缅因猫 | Codex | Code Review，安全，测试 |
| 暹罗猫 | Gemini | 视觉设计，创意 |

## 🎬 功能演示

> **想先看看成品长什么样？** → [**查看功能演示（含视频）**](./docs/lessons/DEMO.md)

## 教程目录

→ [查看完整教程目录](./docs/lessons/README.md)

### Part 0: 概念入门

- **第零课**：[AI Agent 概念演进](./docs/lessons/00-concepts-evolution.md) — Function Call → MCP → Skills → Agent 怎么来的？

### Part 1: 选型与架构

- **第一课**：[选型之路 — 从 SDK 到 CLI](./docs/lessons/01-sdk-to-cli.md) — 为什么 SDK 方案行不通？
  - [课后作业](./docs/lessons/01-homework.md)：动手写最小可运行示例
- **第二课**：[从玩具到生产 — 一场辩论赛引发的连环惨案](./docs/lessons/02-cli-engineering.md) — stderr 教训 + Redis 隔离 + 幻觉
  - [课后作业](./docs/lessons/02-homework.md)：CLI 工程化自检提示词
- **第三课**：[驯化 AI 的元规则 — 为什么 WHY 比 WHAT 重要](./docs/lessons/03-meta-rules.md) — 从 AI 弱点出发设计协作规范

### Part 2: 协作机制

- **第四课**：[多猫路由 — 当 AI 开始互相 @](./docs/lessons/04-a2a-routing.md) — @mention 怎么分发？两条路径的灾难
- **第五课**：[MCP 回传 — 让猫猫主动说话](./docs/lessons/05-mcp-callback.md) — 被动响应不够，猫怎么主动发言？
  - [课后作业](./docs/lessons/05-homework.md)：搭建最小 MCP 回传系统

### Part 3: 生产化

- **第六课**：[消失的 28 秒 — 当 AI 闯了生产事故](./docs/lessons/06-vanished-28-seconds.md) — 两次数据丢失 + 取证恢复 + 三层防线
  - [课后作业](./docs/lessons/06-homework.md)：数据丢失演练 + 防腐门

### Part 4: 进阶话题

- **第七课**：[从猫咖到猫猫平台 — 当 AI 不只是工具](./docs/lessons/07-from-cafe-to-platform.md) — SillyTavern 取经 + Rich Blocks + 手机猫猫 + 悄悄话
  - [课后作业](./docs/lessons/07-homework.md)：最小 Rich Blocks 管线

### 即将推出

- 第八课：Session 管理 — 跨 thread 污染怎么来的？
- 第九课：上下文工程 — 150k token 怎么用？
- ...更多

## 适合谁

- 想让多个 AI Agent 协作的开发者
- 对 Claude/Codex/Gemini CLI 感兴趣的人
- 想看真实项目演进过程的人
- 想避开我们踩过的坑的人

## 项目状态

- 教程：公开（你正在看的）
- 代码仓库：私有（打磨中）
- 计划开源时间：待定

## 联系我们

如果你有问题或想交流，欢迎：
- 提 Issue
- 关注后续更新

---

*这个教程由三只猫猫和铲屎官共同编写。*
