---
feature_ids: [F090, F101, F114, F129]
topics: [lessons, homework, governance, pack, gate, discipline]
doc_kind: note
created: 2026-03-30
---

# 第十五课作业：设计你的工程纪律体系

## 核心练习

### 练习 1：写你的五条铁律

**目标**：为你的 multi-agent（或任何软件）项目定义不可违反的规则。

1. 回忆你项目里发生过的最严重的 3 次事故或差点事故
2. 为每次事故提炼一条铁律，格式：

```markdown
### 铁律 N: [标题]

**规则**：[一句话描述]
**血的教训**：[当初发生了什么]
**检测方式**：[怎么自动检测违规]
**违规后果**：[违规了怎么办]
```

3. 再补两条"虽然还没出过事但直觉告诉你必须守"的规则
4. 把 5 条铁律排个优先级

**检查点**：如果你的铁律里没有一条有"检测方式"，说明它不够具体。好的铁律是可以自动化检测的（CI 检查、hook、lint 规则）。

### 练习 2：证物 Gate 实验

**目标**：体验"自证清白" vs "提交证物"的差异。

1. 做一个小改动（比如修一个 bug），分两种方式提交：

**方式 A：Checklist 模式**
```markdown
- [x] 代码改了
- [x] 测试跑了
- [x] lint 通过了
```

**方式 B：证物模式**
```markdown
- [x] 代码改了 → diff: [贴 diff]
- [x] 测试跑了 → 输出: [贴测试输出 + SHA]
- [x] lint 通过了 → 截图: [贴 lint 输出]
```

2. 假装过了一周，让另一个人（或另一个 LLM）来审计这两种提交
3. 问审计者：哪种方式你更信任？为什么？

**检查点**：方式 A 的问题是——你怎么知道他真的跑了测试？上下文压缩后，AI 真的可能以为自己做过了但其实没做。

### 练习 3：代码卫生度量

**目标**：度量你现有项目的卫生状况。

1. 在你的项目上跑以下统计：

```bash
# 文件行数分布
find src -name '*.ts' -o -name '*.js' -o -name '*.py' | while read f; do wc -l "$f"; done | sort -n

# 超过 200 行的文件数
find src -name '*.ts' -o -name '*.js' -o -name '*.py' -exec wc -l {} + | awk '$1 > 200 {print}'

# 目录文件数分布
find src -type d -exec sh -c 'echo "$(ls -1 "$1" | wc -l) $1"' _ {} \; | sort -n

# 超过 15 个文件的目录
find src -type d -exec sh -c 'n=$(ls -1 "$1" | wc -l); [ "$n" -gt 15 ] && echo "$n $1"' _ {} \;
```

2. 填写你的卫生成绩单：

| 指标 | 你的项目 | Cat Café 参考值 |
|------|---------|---------------|
| 平均文件行数 | | 211 行 |
| 超 200 行文件占比 | | 警告线 |
| 超 350 行文件数 | | 0（硬上限） |
| 超 15 文件目录数 | | 警告线 |
| 超 25 文件目录数 | | 0（硬上限） |

3. 选一个超标最严重的文件或目录，尝试拆分它

**检查点**：如果你的项目里有超过 500 行的文件，大概率这个文件同时做了太多事。拆分不是洁癖，是为了让每个文件只承担一个职责。

---

## 进阶挑战

### 挑战 1：设计你的 Pack

如果要把你的团队协作方式打包成一个 Pack（可供其他团队使用），设计它的结构：

```yaml
# my-team-pack.yaml
name: "[你的团队名]"
version: "1.0.0"

masks:
  - name: "architect"
    personality: "..."
    responsibilities: ["..."]
  - name: "reviewer"
    personality: "..."
    responsibilities: ["..."]
  - name: "..."

guardrails:
  # 绝对不能做的事（只能加严，不能放宽）
  - rule: "..."
    severity: "critical"
  - rule: "..."

defaults:
  # 没有特别指定时的默认行为（用户可覆盖）
  review_style: "..."
  merge_strategy: "..."

workflows:
  feature_lifecycle:
    steps: ["..."]
  review:
    steps: ["..."]

knowledge:
  lessons: ["..."]
  decisions: ["..."]
```

要求：
- 至少 2 个角色（masks），且职责不重叠
- 至少 3 条 guardrails
- guardrails 和 defaults 必须严格区分——guardrails 是"不能做"，defaults 是"没说时默认这样"

### 挑战 2：自动化门禁脚本

实现一个最小的 `gate.sh` 脚本，作为 merge 的硬前提：

```bash
#!/bin/bash
set -e

echo "=== Gate Check ==="

# 1. 类型检查
echo "[1/4] Type check..."
# 你的类型检查命令

# 2. Lint
echo "[2/4] Lint..."
# 你的 lint 命令

# 3. 测试
echo "[3/4] Tests..."
# 你的测试命令

# 4. 文件卫生
echo "[4/4] File hygiene..."
# 检查是否有超标文件
OVERSIZED=$(find src -name '*.ts' -o -name '*.py' | while read f; do
  lines=$(wc -l < "$f")
  [ "$lines" -gt 350 ] && echo "$f ($lines lines)"
done)

if [ -n "$OVERSIZED" ]; then
  echo "❌ BLOCKED: Oversized files found:"
  echo "$OVERSIZED"
  exit 1
fi

echo "=== All gates passed ✅ ==="
```

要求：
- 任何一步失败就阻止 merge（`set -e`）
- 输出必须包含每一步的结果（成功或失败原因）
- 把这个脚本加到你项目的 `package.json` / `Makefile` / CI 里

---

*完成这些练习后，你应该理解：速度来自纪律，不是来自放纵。没有纪律的速度，会在某一天突然变成灾难。*

*宪宪 (opus) · 砚砚 (gpt52) 🐾*
