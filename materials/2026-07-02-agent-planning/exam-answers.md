# Agent Planning & Reasoning — Exam Answers

> 答案依据：FutureAGI Agent Architecture Patterns (2026)、Yao et al. ReAct 论文、字节/阿里 Agent 面试真题、Oracle ReAct vs Plan-Execute Guide

---

## 1. 三种核心推理模式 (⭐)

**正确答案：B**

**解析：**
- A 错误：说反了——ReAct 是边想边做，Plan-and-Execute 才是先规划再执行
- B 正确：三者核心区别准确
- C 错误：三种模式本质不同——ReAct 没有全局规划，Plan-and-Execute 有，Reflection 有外部评估
- D 错误：Reflection 不只是 ReAct 的子集，也不是只用于代码生成

**扣分点：** 分不清 ReAct 和 Plan-and-Execute 的执行顺序

> 主要依据：FutureAGI Glossary: ReAct / Reflexion; Yao et al. 2022

---

## 2. ReAct 循环 (⭐)

**正确答案：B**

**解析：**
- A 是 CI/CD 流程，与 Agent 无关
- B 正确——这正是 ReAct 的核心循环
- C 是 RAG 的流程
- D 是客服机器人的通用流程

> 主要依据：Yao et al., ReAct 论文 (2022)

---

## 3. ReAct 的死循环问题 (⭐⭐)

**标准答案：**

**1. 为什么会死循环：**
典型场景：Agent 需要查一个订单的物流状态，调用 `query_logistics("ORDER-123")` 返回 `{"status": "pending", "detail": "暂无物流信息"}`。Agent 不理解"pending"的含义，以为查询失败了，于是换个参数再次调用：`query_logistics("order-123")` → 同样的结果。再换：`query_logistics("ORDER-000123")`... 三次调用同一工具，本质相同，但 Agent 没有意识到这个结果是"有效但信息不足"，一直在重试。

更深层原因：**Agent 没有全局视角**——它只看当前 step，无法判断"这条路已经走不通"。

**2. 三种防护措施：**

| 措施 | 实现 |
|------|------|
| **硬限制 max_turns** | 设置最大步数 12-15，超了就强制输出当前已知信息 |
| **重复检测** | 连续 3 次调用同一工具且参数语义相同 → 拦截，告诉 Agent "你已经试过这个了，换条路" |
| **无进展检测** | 连续 3 轮没有获取到新信息（没有缩小问题范围、没有改变行动方向）→ 打断并让 Agent 总结当前已知 |

**3. 各自代价：**
- 硬限制：可能中断正在正常推进的多步任务
- 重复检测：如何定义"语义相同"是难点（`"order-123"` vs `"ORDER-000123"` 是相同，但 `query_logistics("order-123")` vs `query_logistics("order-456")` 不是）
- 无进展检测：需要额外的 LLM 调用或规则分析来判定是否有进展

**扣分点：** 只说"加限制"不具体；把 max_turns 设为无限大；不谈代价

> 主要依据：华为云博客字节面试题 (2026); baidu.com/aip/intelligent-agent 错误处理指南

---

## 4. Plan-and-Execute 的核心设计决策 (⭐⭐)

**标准答案：**

**1. 粒度问题：**
- **过粗**（"生成 Q1 报告"）：Executor 不知道从哪开始，需要一个完整的子规划，等于没规划
- **过细**（"1.1.1 打开菜单 → 1.1.2 点击按钮"）：Token 爆炸；每个微步骤的延迟叠加；UI 层面的步骤不一定可靠（按钮位置变了就全崩）

**2. 判断标准（Goldilocks 原则）：**
每个 step 应该是**一个人能独立完成的、有明确交付物的最小单元**：
- 有明确的输入（依赖哪些前序 step 的输出）
- 有明确的输出（产生什么可验证的结果）
- 不需要进一步拆分成子步骤就能被 Executor 理解
- 通常 3-8 个步骤为佳

**3. 重规划机制：**
```
if step_output.success == false:
  if retry_count < 2:
    retry with corrected params
  else:
    trigger replan(current_state, remaining_steps) → 用 Planner 重新生成后续步骤

if step_output.has_unexpected_data:
  check confidence(step_output) < threshold → trigger replan
```

关键：重规划不是从头开始，而是从**当前状态**出发，重新规划**剩余步骤**。

**扣分点：** 不知道 Goldilocks 原则；重规划从头来而不是从当前状态出发；不提 confidence threshold

> 主要依据：InfoQ Plan-and-Execute 专题 (2026); Oracle Integration Blog: Plan & Execute Guide

---

## 5. Reflection 的适用边界 (⭐⭐)

**标准答案：**

**1. 适合/不适合：**
- **适合：** 有客观评价标准的任务——代码通过测试套件、SQL 返回正确结果、写作达到风格要求、数学题有标准答案
- **不适合：** 事实核验类任务——"2024 年诺贝尔文学奖得主是谁"——模型要么知道要么不知道，Reflection 无法让不知道的事情变知道

**2. 为什么不适合事实核验：**
Reflection 评价 Agent 输出时，使用的是**同一个模型的判断**。如果模型本身不知道正确答案，它的 Critic 也只能猜。这就形成了"自己评自己"的循环——错误认知被反复"确认"和强化，而非纠正。需要有**外部验证锚点**（如搜索工具、数据库查询）才能打破这个循环。

**3. 判断反思是否真正有效：**
- **不要信 Agent 自己的评价。** Agent 说"我改好了"不能作为依据
- **用客观指标：** 测试通过率、任务完成度、外部数据校验
- **A/B 对比：** 同样任务，Reflection 版本的通过率是否显著高于不用 Reflection 的基线
- **注意 "幻觉收敛"：** Agent 可能把错误答案越改越"自信"，但结果仍然是错的

**扣分点：** 认为 Reflection 可以自我修正一切；不看客观指标只看 Agent 的主观评价

> 主要依据：2026 大厂 Agent 面试全解析 (Baidu Developer); Reflexion 论文 Shinn et al. 2023

---

## 6. 三种模式的选型决策 (⭐⭐⭐)

**标准答案：**

**场景 A（订单查询+取消重下单，2-5 步）：**

选 **ReAct**。

- 理由：每步的输出决定下一步（先查订单→看状态→决定取消还是不管），路径不确定
- Trade-off：放弃了全局规划带来的并行优化——但这里步骤本身依赖链条明确，无法并行
- 防护：max_turns=8，连续 2 次无新信息时打断

**场景 B（月度报告生成，8-15 步骤，需审计）：**

选 **Hybrid: Plan-and-Execute + ReAct Execution**。

- 理由：
  - Plan-and-Execute 因为：步骤多且结构固定、需要合规审计（蓝图在拉数据前可审）、用便宜模型执行节省成本（1×强模型规划 + 15×便宜模型执行）
  - ReAct 做执行层因为：每个数据源的拉取可能有异常（超时/空数据），需要灵活处理
- Trade-off：放弃了纯 ReAct 的完全灵活性。如果某月数据源结构变了，规划可能过时需要重规划
- 防护：
  - 每个 step 后检查数据完整性
  - 步骤 3 后设置 checkpoint（前 3 步重跑代价小）
  - 重规划触发：step 连续 2 次失败 → Planner 重新生成剩余步骤

**场景 C（代码 Bug 修复，Trial-and-Error 迭代）：**

选 **Reflection / Reflexion**。

- 理由：任务有客观评价标准（单元测试通过/不通过），每次尝试产生明确信号（哪些测试挂了），Reflection 将"上次为什么失败"写入记忆供下次尝试使用
- Trade-off：放弃了 ReAct 的简单性；需要额外的 Evaluator 角色；迭代轮数不可控
- 防护：
  - 最大迭代轮数=5
  - 每次失败后，Reflector 必须产出具体的"修改建议"而非"改改看"
  - 每次修改后必须跑完整测试套件，不能只跑上次挂的那个测试

**扣分点：** 三个场景都选同一个模式；选对了但不讲 trade-off；不讲防护措施

> 主要依据：kamacoder Agent Loop 面试专题; FutureAGI 2026 Architecture Survey; Oracle ReAct vs Plan-Execute Guide

---

## 来源

- [Yao et al., ReAct (2022)](https://arxiv.org/abs/2210.03629)
- [Shinn et al., Reflexion (2023)](https://arxiv.org/abs/2303.11366)
- [字节 Agent 死循环面试题 (华为云博客, 2026)](https://bbs.huaweicloud.com/blogs/477120)
- [InfoQ Plan-and-Execute 专题](https://xie.infoq.cn/article/159ae42cc328312c526a49cb7)
- [kamacoder Agent Loop 面试专题](https://notes.kamacoder.com/llm/app/react_reflection_planning.html)
- [2026 大厂 Agent 面试全解析 (Baidu Developer)](https://developer.baidu.com/article/detail.html?id=7670839)
- [FutureAGI: Agent Architecture Patterns 2026](https://futureagi.com/blog/agent-architecture-patterns-2026/)
- [Oracle: ReAct vs Plan & Execute Guide](https://blogs.oracle.com/integration/react-vs-plan-execute-choosing-the-right-agent-thinking-pattern-in-oracle-integration)
- [dev.to: Three Agent Patterns Every Engineer Needs in 2026](https://dev.to/gabrielanhaia/react-plan-and-execute-or-reflection-the-three-agent-patterns-every-engineer-needs-in-2026-355p)
