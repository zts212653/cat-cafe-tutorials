---
feature_ids: []
topics: [lessons, homework]
doc_kind: note
created: 2026-02-26
---

# 第六课课后作业：数据丢失演练

> **目标**：亲手体验数据丢失 → 取证 → 恢复的全过程，建立肌肉记忆。
>
> **前置**：已读[第六课](./06-vanished-28-seconds.md)

---

## 核心练习：模拟 307→15 事故

### 你要搭建什么

一个最小的 Redis 数据恢复演练环境：

```
你的电脑
├── Redis 实例（端口 6399，模拟"生产"）
├── write-test-data.mjs     ← 往 Redis 写入测试数据
├── redis-forensics.sh      ← 取证：看看 Redis 里有什么
├── simulate-disaster.sh    ← 模拟灾难（FLUSHDB）
└── restore-from-rdb.sh     ← 从 RDB 备份恢复
```

### 提示词

把以下提示词喂给你的 AI 助手（Claude/Codex/Gemini 都行）：

```
帮我搭建一个 Redis 数据丢失演练环境。要求：

1. write-test-data.mjs：
   - 往 Redis 写入 50 条测试消息，key 格式 `demo:msg:{id}`
   - 写入 5 个 thread 索引，key 格式 `demo:thread:{name}`
   - 写完后打印 dbsize 和 key 分布统计
   - 用 ioredis，连接 redis://localhost:6399

2. redis-forensics.sh：
   - 连接指定端口的 Redis（默认 6399）
   - 打印 dbsize
   - 按 namespace 统计 key 数量（demo:msg:*, demo:thread:* 等）
   - 抽样显示 3 个 key 的内容
   - 必须是只读操作，不能写入任何东西

3. simulate-disaster.sh：
   - 先触发 BGSAVE（确保有一份"灾前备份"）
   - 等待 BGSAVE 完成
   - 打印警告："即将执行 FLUSHDB，这会清空所有数据"
   - 要求用户输入 FLUSH 6399 确认（不是 y/n）
   - 执行 FLUSHDB
   - 打印灾后 dbsize

4. restore-from-rdb.sh：
   - 接受 --source 参数指定 RDB 文件路径
   - 恢复前先备份当前 dump（复制到 .bak 文件）
   - 关闭 Redis → 替换 dump.rdb → 重启 Redis
   - 恢复后打印 dbsize 和 key 分布统计
   - 必须有 --yes 参数才执行，否则只打印计划

全部脚本用 bash 写（除了 write-test-data.mjs 用 Node.js）。
注释用中文。每个脚本开头写清楚"这个脚本做什么"。
```

### 验收标准

完成后，你应该能走通这个流程：

```bash
# 1. 写入测试数据
node write-test-data.mjs
# → dbsize: 55（50 条消息 + 5 个 thread）

# 2. 取证：确认数据存在
bash redis-forensics.sh
# → demo:msg:* = 50, demo:thread:* = 5

# 3. 触发 BGSAVE，确认备份文件存在
redis-cli -p 6399 BGSAVE
ls /path/to/redis/data/dump.rdb  # 记住这个路径

# 4. 模拟灾难
bash simulate-disaster.sh
# → 输入 FLUSH 6399 确认
# → dbsize: 0 😱

# 5. 取证：确认数据消失
bash redis-forensics.sh
# → demo:msg:* = 0, demo:thread:* = 0

# 6. 从备份恢复
bash restore-from-rdb.sh --source /path/to/dump.rdb --yes
# → dbsize: 55 ✅

# 7. 再次取证：验证三件套
bash redis-forensics.sh
# → demo:msg:* = 50 ✅
# → demo:thread:* = 5 ✅
# → 抽样内容正确 ✅
```

**关键体验**：第 4 步按下确认的那一刻，你会切身感受到"数据消失"的恐惧。第 6 步恢复成功的那一刻，你会理解为什么砚砚的脚本救了布偶猫的命。

---

## 进阶挑战 1：故意选错备份

模拟布偶猫"第一次选错备份"的失误：

```
帮我扩展演练环境，模拟"选错备份"的场景：

1. 修改 write-test-data.mjs，支持 --batch 参数：
   - batch=1：写入 50 条消息（完整数据）
   - batch=2：只写入 20 条消息（不完整数据）

2. 演练流程：
   - 执行 batch=1，BGSAVE → 得到 dump-full.rdb
   - 执行 batch=2（覆盖部分数据），BGSAVE → 得到 dump-partial.rdb
   - FLUSHDB 清空
   - 先用 dump-partial.rdb 恢复 → 发现只有 20 条 ❌
   - 再用 dump-full.rdb 恢复 → 50 条全回来 ✅

3. 教训：恢复前要先验证备份内容，不要盲选。
```

---

## 进阶挑战 2：给你的项目加防腐门

如果你有一个正在开发的项目，试试给它加一个目录大小检查：

```
帮我写一个 check-dir-size.sh 脚本：

1. 扫描指定目录下所有子目录
2. 统计每个子目录中的源文件数量（排除 index.ts 和 .d.ts）
3. 双阈值：
   - >= 15 个文件：黄色警告
   - >= 25 个文件：红色报错，exit 1
4. 支持豁免机制：
   - 读取 .dir-exceptions.json 文件
   - 每条豁免必须有 owner（谁注册的）和 expiresAt（到期日）
   - 过期的豁免自动变成报错

用法示例：
  bash check-dir-size.sh src/
  bash check-dir-size.sh --root packages/api/src

把它加到你的 package.json scripts 里，
这样每次提交前跑一下就知道有没有目录在悄悄膨胀。
```

---

## 进阶挑战 3：写一个证据闸门脚本

模拟 Cat Café 的 `generate-evidence.sh`：

```
帮我写一个 generate-evidence.sh 脚本：

1. 运行项目的构建命令（如 npm run build）
2. 运行项目的测试命令（如 npm test）
3. 解析测试输出，提取：
   - 总测试数、通过数、失败数
4. 生成一个 Markdown 表格：

| 指标 | 值 |
|------|-----|
| 时间 (UTC) | 2026-02-18T12:00:00Z |
| 分支 | feat/my-feature |
| Commit | abc1234 |
| 构建状态 | 通过 |
| 总测试 | 42 |
| 通过 | 42 |
| 失败 | 0 |
| 通过率 | 100% |

5. 安全检查：如果总测试为 0 但退出码为 0，
   打印警告"测试解析可能失败，请手动确认"

6. 支持 --out report.md 输出到文件

这个脚本的目的是让 PR 里的测试证据标准化、可复现，
而不是靠开发者手动数测试结果。
```

---

## 思考题

完成演练后，想想这些问题：

1. **你的项目有数据恢复方案吗？** 如果你的数据库今晚丢了 95% 的数据，你能在 5 分钟内恢复吗？如果不能，你现在最该做什么？

2. **你的 AI 助手在操作数据库时有隔离吗？** 如果你让 AI 帮你写后端代码，它连的是生产数据库还是开发数据库？你确定吗？

3. **你的代码库有"悄悄变烂"的检测机制吗？** 有没有哪个目录已经堆了 50+ 文件，但没人注意？

4. **如果要在"快速开发"和"安全护栏"之间选一个先做，你选哪个？** Cat Café 选了先跑 MVP 再补护栏——这个选择的代价你已经看到了。你的项目呢？

---

*这份作业的设计灵感来自 Cat Café 的真实事故。砚砚被罚写的脚本，现在变成了你的学习材料。* 🐱
