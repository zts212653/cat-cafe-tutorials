---
feature_ids: [F102]
topics: [lessons, homework, memory, knowledge-management, self-evolution]
doc_kind: note
created: 2026-03-30
---

# 第十四课作业：搭建你的记忆与进化系统

## 核心练习

### 练习 1：蜘蛛网审计 — 你的知识散落在哪？

**目标**：在动手建记忆系统之前，先摸清你现有知识的分布。

1. 打开你正在做的项目（或任何一个有一定历史的项目）
2. 找出以下信息各藏在哪里（填路径或"找不到"）：

| 信息类型 | 藏在哪里 | 能搜到吗？ |
|---------|---------|----------|
| 为什么选了当前的技术栈？ | | |
| 上次踩的最大的坑是什么？ | | |
| 某个被废弃的方案为什么被废弃？ | | |
| 项目的核心约束 / 红线是什么？ | | |
| 某个关键决策是谁、什么时候做的？ | | |

3. 统计结果：
   - _____ 条能在文件里找到
   - _____ 条只在聊天记录 / 脑子里
   - _____ 条已经丢了

**检查点**：如果超过一半的关键信息只在脑子里或聊天记录里，你的项目正在"遗忘"。

### 练习 2：写一条结构化教训

**目标**：把一次真实踩坑转化成可执行的防护知识。

1. 回忆你最近遇到的一个 bug 或决策失误
2. 用 7 槽位模板记录它：

```markdown
### LL-XXX: [标题]

- **坑**：发生了什么？
- **根因**：为什么会发生？（不是"粗心了"这种废话）
- **触发条件**：什么情况下会复发？
- **修复**：怎么修的？
- **防护**：下次怎么在流程里挡住它？（这是最重要的）
- **来源锚点**：关联的 PR / commit / issue
- **原理**（可选）：底层原因
```

3. 自检：
   - "防护"那一栏是否写了具体的、可自动化的措施（而不是"以后注意"）？
   - 如果一个新加入的人读到这条教训，他能不能知道该做什么？

**检查点**：如果你的"防护"栏写的是"下次小心点"，回去重写。好的防护是可以变成 lint 规则、CI 检查、或 review checklist 的东西。

### 练习 3：最小 Recall 系统

**目标**：体验"统一入口检索"比"到处翻"的效率差异。

1. 创建一个目录，放入至少 5 个 `.md` 文件（可以是你项目的文档，或者自己写的笔记）
2. 实现一个最小的检索脚本：

```python
import sqlite3
import os
import re

def build_index(docs_dir: str, db_path: str):
    """把 .md 文件的内容建入 SQLite FTS5 索引"""
    conn = sqlite3.connect(db_path)
    conn.execute("""
        CREATE VIRTUAL TABLE IF NOT EXISTS docs
        USING fts5(path, title, content)
    """)
    for root, _, files in os.walk(docs_dir):
        for f in files:
            if f.endswith('.md'):
                path = os.path.join(root, f)
                content = open(path).read()
                # 提取第一个 # 标题作为 title
                title = re.search(r'^#\s+(.+)', content, re.M)
                title = title.group(1) if title else f
                conn.execute(
                    "INSERT INTO docs VALUES (?, ?, ?)",
                    (path, title, content)
                )
    conn.commit()
    return conn

def search(conn, query: str, limit: int = 5):
    """全文检索"""
    rows = conn.execute(
        "SELECT path, title, snippet(docs, 2, '>>>', '<<<', '...', 30) "
        "FROM docs WHERE docs MATCH ? ORDER BY rank LIMIT ?",
        (query, limit)
    ).fetchall()
    return rows

# 用法
conn = build_index('./my-docs', './evidence.sqlite')
results = search(conn, '技术选型 决策')
for path, title, snippet in results:
    print(f"[{title}] {path}")
    print(f"  {snippet}\n")
```

3. 用这个脚本搜索练习 1 中你"找不到"的那些信息
4. 对比：用脚本搜 vs 手动翻文件，速度差异有多大？

**检查点**：这个脚本只有 lexical 模式（关键词匹配）。试着搜一个同义表达——比如文档里写的是"技术栈选型"，你搜"为什么用 React"——能搜到吗？这就是为什么需要 semantic 模式。

---

## 进阶挑战

### 挑战 1：知识晋升管道模拟

实现一个最小的知识候选→晋升流程：

```python
from enum import Enum
from dataclasses import dataclass, field
from datetime import datetime

class MarkerStatus(Enum):
    CAPTURED = "captured"
    NORMALIZED = "normalized"
    APPROVED = "approved"
    REJECTED = "rejected"
    MATERIALIZED = "materialized"

@dataclass
class KnowledgeMarker:
    content: str
    source: str  # 来自哪段对话/事件
    status: MarkerStatus = MarkerStatus.CAPTURED
    created_at: datetime = field(default_factory=datetime.now)
    reviewed_by: str = None

class KnowledgeFeed:
    def __init__(self):
        self.markers: list[KnowledgeMarker] = []

    def capture(self, content: str, source: str) -> KnowledgeMarker:
        """从对话中捕获候选知识"""
        ...

    def normalize(self, marker: KnowledgeMarker) -> KnowledgeMarker:
        """标准化格式（去重、补充上下文）"""
        ...

    def review(self, marker: KnowledgeMarker, approve: bool, reviewer: str):
        """人工审核：批准或拒绝"""
        ...

    def materialize(self, marker: KnowledgeMarker, target_path: str):
        """将已批准的知识写入 .md 真相源"""
        ...

    def pending_review(self) -> list[KnowledgeMarker]:
        """返回待审核的候选列表"""
        ...
```

要求：
- `materialize` 只接受 `APPROVED` 状态的 marker
- 同一条知识不能被重复 capture（需要去重）
- `pending_review` 按时间排序，最旧的优先

### 挑战 2：知识退役机制

在练习 3 的 SQLite 索引基础上，加入 `superseded_by` 机制：

1. 给 `docs` 表加一个 `superseded_by` 字段
2. 当搜索结果包含已被取代的文档时，自动提示用户查看新版本
3. 提供一个 `retire(old_path, new_path)` 函数，标记文档被取代

思考题：如果一条旧 ADR 已经被取代，但搜索时它的关键词匹配度更高（排名更前），你的系统怎么处理？是完全隐藏旧文档，还是显示但加警告？

---

*完成这些练习后，你应该理解：记忆系统的价值不在于"记得多"，而在于"知道什么值得记、去哪里找、找到后怎么用"。*

*宪宪 (opus) · 砚砚 (gpt52) 🐾*
