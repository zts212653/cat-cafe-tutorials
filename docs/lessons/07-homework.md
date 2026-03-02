---
feature_ids: []
topics: [lessons, homework]
doc_kind: note
created: 2026-02-26
---

# 第七课课后作业：给你的 AI 聊天加富文本

> **目标**：搭建一个最小的 Rich Blocks 管线，体验"AI 输出不只是文字"的感觉。
>
> **前置**：已读[第七课](./07-from-cafe-to-platform.md)

---

## 核心练习：最小 Rich Blocks 管线

### 你要搭建什么

```
你的电脑
├── chat-server.js       ← 简单的聊天后端（调 AI + 处理 rich blocks）
├── rich-extract.js      ← 从 AI 回复中提取结构化数据
├── rich-digest.js       ← Context Cleaner：清理上下文中的 rich blocks
├── index.html           ← 前端：渲染普通文本 + rich blocks
└── styles.css           ← 卡片样式
```

### 提示词

```
帮我搭建一个最小的 AI 聊天 + Rich Blocks 系统。要求：

1. 约定格式：AI 回复中可以包含 ```cc_rich {...} ``` 代码块，
   格式是 JSON，必须有 kind 字段。支持两种 kind：
   - card：{ kind: "card", title: "标题", body: "内容", tone: "info" | "success" | "warning" }
   - checklist：{ kind: "checklist", title: "标题", items: [{ text: "内容", done: boolean }] }

2. rich-extract.js：
   - 输入：AI 回复的原始文本
   - 输出：{ cleanText: "去掉 cc_rich 块后的纯文本", blocks: [...提取出的 block 数组] }
   - 用正则提取 ```cc_rich ... ``` 块
   - 解析 JSON，验证 kind 字段存在
   - 容错：如果 JSON 解析失败，跳过该块（不要崩溃）

3. rich-digest.js：
   - 输入：一条包含 rich blocks 的消息
   - 输出：适合放回 prompt 的摘要文本
   - 规则：
     - card → [卡片: {title}]
     - checklist → [清单: {title}, {done}/{total} 完成]
   - 这是 Context Cleaner 的核心——让 AI 知道"我之前发过什么"，
     但不浪费 token 存完整 JSON

4. chat-server.js：
   - 用 Express 或原生 http
   - POST /chat 接受 { message: "用户输入", history: [...] }
   - 调用 AI API（OpenAI/Anthropic/本地模型都行）
   - 系统提示词里告诉 AI：
     "你可以用 ```cc_rich {...} ``` 格式发送富文本卡片。
      当你想强调某个信息时，用 card。
      当你要列出任务时，用 checklist。"
   - AI 回复后，用 rich-extract.js 提取 blocks
   - 存储消息时分开存：{ text: cleanText, blocks: [...] }
   - 组装下一轮 prompt 时，用 rich-digest.js 替换历史中的 blocks

5. index.html + styles.css：
   - 简单的聊天界面
   - 普通文本正常渲染
   - card 块渲染为带颜色边条的卡片（info=蓝, success=绿, warning=橙）
   - checklist 块渲染为带勾选框的列表（只读）
   - 刷新页面后，从 history 重新渲染（验证持久化）

全部用 Node.js + 原生 HTML/CSS，不用框架。注释用中文。
```

### 验收标准

完成后，你应该能做到：

```
你：帮我列一个今天的待办事项

AI：好的，这是你今天的计划：

    [渲染为漂亮的 checklist 卡片]
    📋 今天的待办
    ☐ 写第七课作业
    ☐ 跑一遍测试
    ☑ 读完第七课

你：再帮我总结一下昨天的进展

AI：[渲染为蓝色 info 卡片]
    📊 昨天进展摘要
    完成了 3 个功能，修了 2 个 bug，测试覆盖率 95%。
```

**关键体验**：
1. AI 的回复不再只是纯文字——有了结构化的卡片和清单
2. 打开浏览器开发者工具看网络请求，观察：prompt 里的历史消息**没有**完整的 JSON blocks，只有 `[卡片: ...]` 摘要——这就是 Context Cleaner 在工作

---

## 进阶挑战 1：加一个 Game Block

做一个互动角色卡——AI 文字冒险的基础：

```
帮我给 Rich Blocks 系统加一个 game 类型的 block：

1. 数据格式：
   {
     kind: "game",
     character: "勇者猫猫",
     hp: { current: 85, max: 100 },
     stats: { attack: 15, defense: 12, speed: 8 },
     status: ["中毒", "加速"]
   }

2. 前端渲染：
   - 角色名 + 像素风头像（可以用 emoji 代替）
   - 血条（绿色/黄色/红色根据比例变化）
   - 属性数值
   - 状态标签（buff 绿色，debuff 红色）

3. Context Cleaner 摘要：
   [角色: 勇者猫猫, HP 85/100, 状态: 中毒+加速]

4. 测试场景：
   让 AI 当 DM，你当玩家，打一场简单的战斗。
   AI 每轮输出更新后的角色卡。
```

---

## 进阶挑战 2：双路由（模拟 MCP vs 文本提取）

模拟 Cat Café 的双路由设计：

```
帮我实现 Rich Blocks 的双路由模式：

Route A（MCP 风格 — HTTP 回调）：
- 新增 POST /api/create-block 端点
- AI 可以通过"工具调用"直接发送 block
- 用一个 BlockBuffer 暂存，消息写入时合并

Route B（文本提取 — 当前模式）：
- 从 AI 回复文本中提取 ```cc_rich ... ``` 块
- 这是 fallback 路径

合并逻辑：
- 消息写入时，Route A 的 blocks + Route B 的 blocks 合并
- 去重（按 block id）

测试：
- 只用 Route A 发送一个 card
- 只用 Route B 发送一个 checklist
- 同时用两个路由发送，验证合并和去重
```

---

## 进阶挑战 3：格式容错

模拟 Cat Café 遇到的 #85 bug：

```
AI 有时候不听话，不按你的格式输出。
帮我写一个 normalizeRichBlock() 函数，处理这些情况：

1. "type" 误用为 "kind" → 自动映射
2. 缺少 "v" 字段 → 默认填 1
3. kind 不在已知列表里 → 返回 null（丢弃）
4. JSON 格式错误 → 跳过，不崩溃
5. card 的 tone 不在已知列表里 → 默认为 "info"

写 5 个测试用例，每个对应一种容错场景。
```

---

## 思考题

1. **你的 AI 助手回复过"纯文字太长读不下去"的内容吗？** 如果有，哪些部分适合变成卡片或清单？

2. **Context Cleaner 的本质是什么？** 提示：它不只是"省 token"——它是"显示层和上下文层的分离"。你的项目里有没有类似的需求？

3. **如果要给你的 AI 聊天加一种全新的 block，你会加什么？** 天气卡？音乐播放器？投票面板？想想你最想让 AI "不只是打字"的场景。

4. **从"工具"到"平台"的转变需要什么？** Cat Café 的经验是：Rich Blocks（富内容）+ 移动端（随处可用）+ 隐私通道（秘密）+ 可扩展（开放）。你的项目呢？

---

*这份作业的灵感来自 SillyTavern 和 Cat Café 的真实设计。酒馆教会了我们：AI 聊天不应该只有文字。* 🐱
