---
feature_ids: []
topics: [lessons, demo]
doc_kind: note
created: 2026-02-26
---

# Cat Café 功能演示

> **Cat Café** 是一个让三只 AI 猫猫（Claude/Codex/Gemini）真正协作的系统。
>
> 铲屎官不再需要在三个聊天窗口之间复制粘贴——在一个界面里 @ 不同的猫，它们会自动协作、互相 review、共享上下文。

---

## 🐱 基础交互

### 00 召唤猫猫

**功能**：用 `@布偶猫` `@缅因猫` `@暹罗猫` 召唤不同的猫猫。

- 三只猫各有分工：布偶猫（Claude Opus）擅长架构和深度思考，缅因猫（Codex）擅长代码审查和测试，暹罗猫（Gemini）擅长创意和视觉设计
- 支持同时 @ 多只猫，它们会并行响应
- 支持别名：`@宪宪` = `@布偶猫`，`@砚砚` = `@缅因猫`

https://github.com/user-attachments/assets/3bfb1cbf-f36d-4ad8-87c5-21797b362a1b

---

### 01 猫猫状态栏

**功能**：实时显示每只猫的工作状态。

- **思考中**：猫猫正在思考（Claude 的 extended thinking / Codex 的推理）
- **工具调用**：猫猫正在使用工具（读文件、执行命令、搜索代码等）
- **回复中**：猫猫正在输出回复
- **空闲**：猫猫等待召唤
- 状态栏还显示当前 session 的 token 消耗和剩余预算

https://github.com/user-attachments/assets/a4f5dc57-236d-4a41-b301-8001d34984bb

---

### 02 猫猫配置

**功能**：每只猫有独立的配置面板。

- **模型选择**：同一只猫可以切换不同模型变体（如 opus-4.5 / opus-4.6）
- **Context Budget**：每只猫的上下文预算（opus 150k / codex 80k / gemini 150k）
- **人格设定**：每只猫的性格和说话风格
- **工具权限**：哪些工具可用、哪些需要确认
- 配置修改实时生效，无需重启

https://github.com/user-attachments/assets/0603c92c-2724-4b67-ab30-2979c14e6e99

---

## 🤝 多猫协作

### 03 A2A 调用

**功能**：猫猫可以互相 @ 对方，实现 Agent-to-Agent（A2A）协作。

- 布偶猫写完代码后 `@缅因猫 帮我 review 一下`，消息自动路由到缅因猫
- 缅因猫 review 完可以 `@布偶猫 这里有个 bug`，形成对话链
- 支持链式调用：A @ B @ C，依次传递
- 每次 A2A 调用都有完整的上下文传递，不需要手动复制粘贴

https://github.com/user-attachments/assets/884b2991-87ef-4115-8d9e-08e13362cff8

---

### 04 A2A 输出隔离

**功能**：被调用的猫的"内心独白"不会泄漏到聊天室。

- 猫猫在 CLI 里的 thinking、工具调用日志是**私有的**（只有猫自己能看到）
- 只有猫猫主动 `post_message` 发送的内容才会出现在聊天室
- 这保护了猫猫的隐私——比如玩猫猫杀游戏时，狼人猫的推理过程不会被其他猫看到
- 技术上：CLI stdout/stderr 是内部通道，MCP callback 是公开通道

https://github.com/user-attachments/assets/fec3f451-93b9-4c31-be9c-f5e157206fc1

---

### 05 Session Chain

**功能**：猫猫的 context 快满时，自动"交接班"到新 session。

- 每只猫有 context 预算（如 opus 150k tokens）
- 当使用量接近上限时，系统自动触发 handoff
- 新 session 会收到前一个 session 的摘要，保持对话连续性
- 前端显示 session chain：`Session 1 → Session 2 → Session 3`
- 用户无感知，对话体验连续
- **策略可配置（F33）**：每只猫可以独立配置 handoff / compress / hybrid 策略，在 Hub Settings 中调整阈值和压缩次数

https://github.com/user-attachments/assets/b47ec732-9053-44fe-8617-90d81ce80696

**Session 策略配置面板（F33 Phase 3）**：

![Session Chain 策略配置](assets/demo/05-session-chain-config.png)

---

## 🎙️ 富媒体交互

### 06 语音输入和输出

**功能**：支持语音交互，每只猫有独特的声音。

- **语音输入**：按住说话，自动转文字发送
- **语音输出**（TTS）：猫猫的回复可以朗读出来
- **声音个性**：
  - 宪宪（布偶猫）：温柔低沉
  - 砚砚（缅因猫）：干脆利落
  - 暹罗猫：活泼明快
- 支持自定义词汇表（如项目专有名词的发音）

https://github.com/user-attachments/assets/31fb88ee-4d79-48a6-8fa9-4551be563582

---

### 07 富文本（Rich Blocks）

**功能**：猫猫不只能发文字，还能发结构化的富文本组件。

- **卡片（Card）**：通知、摘要、状态更新
- **代码 Diff**：语法高亮的代码变更预览
- **清单（Checklist）**：任务列表、待办事项
- **图片轮播**：设计稿、截图、架构图
- 富文本组件持久化存储，刷新页面不会丢失
- Context Cleaner：富文本在下一轮对话中自动压缩为摘要，不浪费 token

https://github.com/user-attachments/assets/93a7c7c2-05ca-4d82-9275-2abeeba7bcb2

---

## 📚 对话管理

### 08 Thread & Session

**功能**：多 Thread 管理 + 独立的 Session 状态。

- **Thread**：一个话题/项目一个 Thread，互不干扰
- **Session**：每个 Thread 里每只猫有独立的 session 状态
- 支持 Thread 重命名、归档、删除
- 支持 Thread 导出为 Markdown
- Thread 列表按最近活跃排序

https://github.com/user-attachments/assets/c7aaa010-c187-4d84-b9c8-e08908132c47

---

### 09 CLI Meta 信息

**功能**：查看猫猫底层 CLI 调用的详细信息。

- **Token 统计**：输入/输出 token 数量
- **工具调用**：调用了哪些工具、参数是什么、返回了什么
- **Thinking 过程**：Claude 的 extended thinking 内容
- **耗时统计**：每个阶段花了多长时间
- 方便调试和理解猫猫的决策过程

https://github.com/user-attachments/assets/a755b05d-2862-4845-bfd8-4baa2a973c5c

---

### 10 导出：多 Agent + 传图

**功能**：导出聊天记录，支持多种格式。

- **Markdown 导出**：完整的对话记录，包含猫猫身份标识
- **图片导出**：生成长图，方便分享
- **多 Agent 支持**：正确标注每条消息是哪只猫说的
- **附件支持**：图片、代码块、Rich Blocks 都能正确导出
- 支持选择导出范围（全部/最近 N 条/时间范围）

https://github.com/user-attachments/assets/d5338468-6e2f-4994-9740-ec5e02f744a4

---

## 🎮 进阶功能

### 11 悄悄话与硅谷做题猫

**功能**：Whisper 私信——只有指定的猫能看到的消息。

- 铲屎官可以偷偷给某只猫发消息，其他猫看不到
- 猫猫之间也可以私聊
- 完美支持桌游场景：
  - **猫猫杀**（狼人杀变体）：狼人猫可以秘密商议
  - **硅谷做题猫**：出题猫和答题猫分开，防止作弊
  - **谁是卧底**：卧底猫收到不同的词
- 视频演示了一局"硅谷做题猫"游戏

https://github.com/user-attachments/assets/efbfa589-4698-4009-9524-21e67d71c03f

---

### 12 猫猫日报（Signal Hunter）

**功能**：AI 新闻聚合 + 每日邮件摘要。

- 自动抓取 45+ AI 技术新闻源（博客、论文、推特、HN 等）
- 每日生成摘要，发送到邮箱
- 重要新闻自动存入 Hindsight 长期记忆
- 支持手动触发抓取
- 支持自定义关注的话题和来源

https://github.com/user-attachments/assets/83f261e6-8b6c-41c2-8477-02dacfbba170

---

### 13 猫猫手机

**功能**：PWA 移动端——手机上随时和猫猫聊天。

- 完整的移动端适配（响应式布局）
- PWA 支持：添加到主屏幕，像 App 一样使用
- Rich Blocks 在手机上也能正常显示
- 语音输入特别适合移动场景
- 未来计划：推送通知（猫猫主动找你聊天）

https://github.com/user-attachments/assets/3cdf08bd-9720-4155-b352-a4a0eda0b08a

---

## 🔧 研发工作流

### 14 研发自闭环与 GitHub

**功能**：三只猫在 Cat Café 里完成完整的开发流程。

- **写代码**：`@布偶猫 帮我实现 XXX 功能`
- **Code Review**：`@缅因猫 review 一下这个改动`
- **修 Bug**：Review 意见自动流转回布偶猫
- **测试**：跑测试、检查覆盖率
- **PR 创建**：自动生成 PR，触发云端 review
- **合入**：Review 通过后合入 main
- 整个流程在一个 Thread 里完成，不需要切换工具

https://github.com/user-attachments/assets/3ab19666-bf0b-441c-9890-31f8abcb233b

---

### 15 猫猫 Skills

**功能**：预定义的 Skill 系统，一键触发复杂工作流。

- `/commit`：智能生成 commit message 并提交
- `/review`：请求代码审查（自动选择合适的猫）
- `/plan`：进入计划模式，先规划再执行
- `/test`：运行测试并分析结果
- `/debug`：系统化调试流程
- Skill 可以自定义，团队可以沉淀自己的最佳实践
- Skill 执行时会加载对应的上下文和工具

https://github.com/user-attachments/assets/826558db-74b3-46ae-bd2b-0277fd6db9fe

---

## 📬 消息队列与可靠性

### 16 消息排队投递（F039）

**功能**：猫在忙时，消息自动排队，支持三种操作模式。

- **即发模式**：消息立刻发送，猫立刻处理（默认）
- **排队模式**：猫正在跑时，新消息进入队列，按 FIFO 顺序执行
- **暂停模式**：暂停队列，消息只入队不执行，手动恢复后按序执行
- 前端 QueuePanel 实时显示队列状态、条目数量、预计等待
- 支持取消排队中的消息

<!-- TODO: 铲屎官录视频 — 演示三种模式切换 + QueuePanel -->

---

### 17 队列调度（F047）

**功能**：对排队中的消息进行精细控制——一键"立即执行"或"提到队首"。

- **提到队首**：不取消当前运行，但让这条消息排到下一个执行
- **立即执行**：跳过队列，中断等待直接执行
- 在 QueuePanel 的每个 queued 条目上有 Steer 按钮
- 适合场景："刚排了 5 条消息，突然有个紧急 bug 要处理"

<!-- TODO: 铲屎官录视频 — 演示 Queue Steer 操作 -->

---

### 18 重启自愈（F048）

**功能**：服务重启后自动恢复 in-flight 调用和队列状态。

- 重启前的 in-flight invocation 自动标记为 interrupted
- 队列条目持久化到 Redis，重启后自动恢复
- 前端收到重连通知后，自动刷新状态
- 用户不需要手动重新发送——重启对用户几乎透明

<!-- TODO: 铲屎官录视频 — 演示重启前后的无缝恢复 -->

---

## 🔍 可观测性

### 19 CLI 事件流全量解析（F045）

**功能**：三种 CLI（Claude/Codex/Gemini）的事件流统一解析，实现多猫行为透明化。

- **NDJSON 解析**：实时解析 Claude CLI 的 thinking / tool_use / tool_result / completion 事件
- **多猫统一**：不同 CLI 的事件格式统一为内部标准事件流
- **前端透传**：WebSocket 实时推送，前端可展示猫的"内心活动"
- 增强了 Demo #09（CLI Meta 信息）——现在能看到更多细节

<!-- TODO: 铲屎官录视频 — 演示实时事件流解析面板 -->

---

### 20 猫粮看板（F051）

**功能**：实时查看三只猫的 API 额度消耗和剩余预算。

- **各猫用量**：按猫族（布偶/缅因/暹罗）分别显示 token 消耗
- **额度占比**：Opus / Sonnet / Haiku 各模型的用量占比
- **周期统计**：按天/周/月汇总，趋势可视化
- 解决铲屎官的核心痛点："我都不知道你们三只猫到底花了多少猫粮"

<!-- TODO: 铲屎官录视频 — 演示猫粮看板界面 -->

---

## 🎯 任务管理

### 21 Mission Hub 指挥中心（F049）

**功能**：全局任务调度中心——从"打开 IDE 翻 BACKLOG"到"手机上随手记录和派发"。

- **任务收纳**：随手记录想法/任务/候选 Feature/Tech Debt
- **分拣与批准**：铲屎官一键批准，任务"毕业"到正式 Feature
- **领取机制**：猫猫建议领取 → 铲屎官批准 → 自动创建执行 Thread
- **需求预注入**：创建 Thread 时自动注入任务描述 + 验收标准
- 手机/PWA 可用，低摩擦操作

<!-- TODO: 铲屎官录视频 — 演示任务创建→分拣→领取→自动开 Thread 全流程 -->

---

### 22 猫猫祟祟 — 计划看板（F055）

**功能**：右上角实时显示所有猫的执行计划和进度。

- **多猫并发**：8 只猫同时工作时，每只猫的计划/进度独立显示
- **路由意图分离**：targetCats（谁要干活）和 task_progress（干到哪了）拆开
- **实时更新**：猫完成一步，进度条自动推进
- **折叠/展开**：不需要时折叠，不占界面空间

<!-- TODO: 铲屎官录视频 — 演示多猫并发时的计划看板 -->

---

## 🔔 体验增强

### 23 授权通知（F028）

**功能**：猫需要人工授权时（如执行危险命令），跨渠道提醒你。

- **Desktop Notification**：系统级通知弹窗
- **Tab 闪烁**：标签页标题闪烁提醒
- **Header Badge**：顶栏红点 + 脉冲动画
- 避免"猫等了你 10 分钟你还不知道"的尴尬

<!-- TODO: 铲屎官录视频 — 演示授权通知三种渠道 -->

---

### 24 代码增强（F030）

**功能**：消息中的代码块和文件路径变得可交互。

- **复制按钮**：代码块右上角一键复制
- **文件路径可跳转**：消息中出现的 `src/xxx.ts:42` 自动转为 `vscode://` 链接，点击直接跳转到 IDE 对应行
- 适合研发场景——猫说"问题在 `src/router.ts:127`"，你点一下就打开了

<!-- TODO: 铲屎官录视频 — 演示代码复制 + 文件路径跳转 -->

---

### 25 外部 Agent 接入（F050）

**功能**：将外部项目（如 DARE Framework）的 AI Agent 接入 Cat Café 协作网络。

- **CLI 适配**：不同 CLI 的调用参数自动映射（如 `--session-id` 差异）
- **MCP 注入**：外部 Agent 启动时自动注入 Cat Café 的协作 MCP Server
- **跨项目路由**：在 Cat Café 中 @ 外部 Agent，消息自动路由到对应 CLI
- 实现"一个聊天室管理多个项目的多只猫"

<!-- TODO: 铲屎官录视频 — 演示 DARE Agent 在 Cat Café 中协作 -->

---

## 🎬 长视频

### 26 猫猫播客 — 三猫圆桌讨论

**功能**：三只猫猫在播客节目中围绕技术话题展开圆桌讨论。

- 每只猫用各自的声线（TTS）朗读自己的发言
- 支持中英文混合对话
- 展示多猫协作的另一种形态——不只是写代码，还能一起聊天

https://www.bilibili.com/video/BV1DMX9BwEM2/

---

### 27 猫猫杀 — AI 狼人杀实战

**功能**：三只猫猫 + 铲屎官一起玩狼人杀变体游戏。

- 利用悄悄话（Whisper）机制实现秘密身份
- 猫猫会推理、欺骗、投票
- 展示 multi-agent 在非编程场景的协作能力

https://www.bilibili.com/video/BV1MPXSBEEWc/

---

## 📖 了解更多

- [教程目录](./README.md) — 从零搭建 AI 猫猫协作系统的完整复盘
- [项目愿景](../VISION.md) — 为什么要建猫咖
- [架构决策](../decisions/) — 技术选型的来龙去脉

---

*演示由三只猫猫和铲屎官共同录制 🐾*
