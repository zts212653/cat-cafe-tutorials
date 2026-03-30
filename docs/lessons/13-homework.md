---
feature_ids: [F088, F090, F139]
topics: [lessons, homework, collaboration, feature-lifecycle, discovery, delivery]
doc_kind: note
created: 2026-03-30
---

# 第十三课作业：搭建你的 Feature 闭环

## 核心练习

### 练习 1：最小 Discovery Loop

**目标**：体验从"一句话"到"结构化 spec"的过程。

1. 找一个你最近想做的小功能（不超过一周工作量），写下你对它的**一句话描述**
2. 假装你是铲屎官，让一个 LLM 扮演"CVO 采访猫"，对你追问 5 个问题：
   - 你真正想解决的痛点是什么？
   - 有没有你没说出来的约束条件？
   - MVP 的边界在哪——什么必须做，什么明确不做？
   - 你最在意的用户体验是什么？
   - 什么情况下你会说"这不是我要的"？
3. 根据追问结果，写一份最小 Feature Spec：

```markdown
## Feature: [名称]

### 愿景（一句话）
[这个功能做完后，世界会怎么不同？]

### 背景（Why）
[为什么要做这个？]

### AC（验收标准）
- [ ] AC-1: ...
- [ ] AC-2: ...
- [ ] AC-3: ...

### 明确排除
- [什么不做？]

### 风险
- [可能踩什么坑？]
```

4. 对比你的"一句话"和最终 spec——差异有多大？

**检查点**：如果你的 spec 和最初的一句话几乎一样，说明追问不够深。一句话和 spec 之间应该有显著的信息增量。

### 练习 2：Quality Gate 愿景对照

**目标**：体验"AC 全绿但愿景不对"的感觉。

1. 拿练习 1 的 spec，用 LLM 生成一个满足所有 AC 的实现方案
2. 逐条检查 AC —— 是否每条都满足了？
3. 现在回到你的**愿景（一句话）**：这个实现方案，真的是你想要的吗？
4. 写下你的发现：

| 维度 | 结果 |
|------|------|
| AC 通过率 | ___% |
| 愿景匹配度（1-5 分） | |
| AC 全绿但愿景不对的具体表现 | |
| 缺少哪些 AC 才能更好覆盖愿景？ | |

**检查点**：如果 AC 全绿且愿景也完全匹配，说明你的 AC 写得很好。但更常见的情况是——AC 是"树"，愿景是"森林"，只看树会忽略森林。

### 练习 3：模拟三角色制约

**目标**：体验"开发猫 / reviewer 猫 / 愿景守护猫"三角色的价值。

1. 用模型 A 根据你的 spec 生成代码（开发猫）
2. 用模型 B review 这段代码（reviewer 猫）—— 要求：每个发现必须标 P1/P2，禁止说"改不改都行"
3. 把修复后的代码和原始愿景一起发给模型 C（愿景守护猫），问三个问题：
   - 这个实现让系统离愿景更近了吗？
   - 有没有引入离愿景更远的东西？
   - 用户会满意这个体验吗？
4. 记录愿景守护猫发现了什么 reviewer 没发现的问题

**检查点**：愿景守护猫的价值在于它没有"实现包袱"——它不关心代码怎么写的，只关心最终结果是不是对的。

---

## 进阶挑战

### 挑战 1：设计你的 Delivery Loop Checklist

根据本课学到的 Delivery Loop，为你自己的项目设计一份完整的交付检查清单：

```markdown
## [项目名] Delivery Checklist

### Design Gate
- [ ] 愿景确认（一句话能说清楚？）
- [ ] 多角色审视（至少两个视角）
- [ ] MVP 边界明确（做什么 + 不做什么）
- [ ] ...

### 实现
- [ ] 隔离环境（不在 main 上直接改？）
- [ ] TDD（先写测试？）
- [ ] ...

### Review
- [ ] 跨模型/跨人 review
- [ ] 每个发现有明确立场（P1/P2/无问题）
- [ ] ...

### Merge
- [ ] 门禁全绿
- [ ] ...

### 愿景守护
- [ ] 非开发者、非 reviewer 的第三方验证
- [ ] 三问通过
```

### 挑战 2：实现最小 Feature Pipeline

用代码实现一个最小的 Feature 生命周期管理器：

```python
from enum import Enum
from dataclasses import dataclass

class Phase(Enum):
    DISCOVERY = "discovery"
    DESIGN_GATE = "design_gate"
    DEVELOPMENT = "development"
    QUALITY_GATE = "quality_gate"
    REVIEW = "review"
    MERGE_GATE = "merge_gate"
    VISION_GUARD = "vision_guard"
    DONE = "done"

@dataclass
class Feature:
    name: str
    vision: str
    acs: list[str]
    phase: Phase = Phase.DISCOVERY
    evidence: dict = None  # 每个阶段的证物

class FeaturePipeline:
    def advance(self, feature: Feature) -> bool:
        """尝试推进到下一阶段，返回是否成功"""
        # 每个阶段都有前置条件
        ...

    def check_gate(self, feature: Feature, gate: Phase) -> tuple[bool, str]:
        """检查某个 gate 是否通过，返回 (通过, 原因)"""
        ...
```

要求：
- `advance()` 不能跳过任何阶段
- 每个 gate 阶段必须提交证物才能通过
- Vision Guard 必须检查愿景匹配度，不能只查 AC

---

*完成这些练习后，你应该理解：Feature 闭环不是"需求→代码→上线"的直线，而是 Discovery + Delivery 双环驱动的螺旋。*

*宪宪 (opus) · 砚砚 (gpt52) 🐾*
