# Agent Planning & Reasoning

## 1. 定义 / 背景

**Agent Planning（智能体规划）** 是 Agent 系统的决策核心——决定"How（采用什么策略）"和"When（何时采取下一步）"。没有规划的 Agent 只是调用工具的聊天机器人。[来源: dev.to 2026 Agent Patterns Survey]

### 为什么需要规划

- **不确定性管理：** 用户需求往往模糊，Agent 需要把模糊目标分解成可执行的步骤
- **异常恢复：** 工具失败、返回意外结果时，Agent 需要动态调整路径而不是崩溃
- **资源约束：** Token 预算、时间预算、工具调用次数都有上限，需要全局优化而非局部贪心 [来源: FutureAGI Agent Architecture Patterns 2026]

### 三种核心模式

| 模式 | 一句话概括 | 代表论文 |
|------|-----------|---------|
| **ReAct** | 边想边做：Thought → Action → Observation 循环 | Yao et al. 2022 |
| **Plan-and-Execute** | 先规划再执行：规划器出蓝图 → 执行器逐步跑 | Wang et al. 2023 |
| **Reflection / Reflexion** | 做完回头看：执行 → 评估 → 反思 → 重试 | Shinn et al. 2023 |

[来源: FutureAGI Glossary: ReAct, 2026]

---

## 2. 核心原理 / 架构 / 机制

### 2.1 ReAct（Reasoning + Acting）

ReAct 的核心是一个**紧密的 Thought → Action → Observation 循环**：

```
User: "帮我退掉 #4521 号订单"
Thought: 需要先查下这个订单
Action: search_orders("4521")
Observation: 订单金额 ¥623，状态"已完成"，可退款
Thought: 订单有效，执行退款
Action: refund_order("4521")
Observation: 退款成功，退款单号 RF-8891
Thought: 可以给出最终回复了
Final Answer: 已为您退款 ¥623，退款单号 RF-8891
```

**关键特征：**
- Agent 没有完整规划，每一步根据上一步的工具输出决定下一步
- 所有 Thought 和 Observation 保留在上下文中，形成推理链
- 灵活但容易"漂移"——步骤多了可能偏离原始目标

[来源: ReAct论文 Yao et al. 2022; blog.n8n.io ReAct Agent Guide]

### 2.2 Plan-and-Execute（先规划再执行）

将 Agent 拆成两个角色：

```
Planner（强模型, 如 GPT-5 / Claude Opus 4.7）:
  → 分析目标 → 输出步骤列表 [Step1, Step2, ..., StepN]

Executor（便宜模型, 如 GPT-4o-mini / Haiku）:
  → 按顺序执行每个 Step → 必要时触发重规划
```

**关键设计决策：**

1. **规划粒度：** 太粗→执行器无从下手；太细（嵌套 1.1/1.2）→ Token 爆炸。最佳实践是 3-8 个高层级步骤 [来源: InfoQ Plan-and-Execute 专题 2026]
2. **重规划触发：** 每 K 步后检查，或当 step 输出超出置信阈值时
3. **Sandbox 隔离：** 每个 step 在隔离上下文中执行，避免前一步的错误污染后续

[来源: Oracle Integration Blog: ReAct vs Plan & Execute 2026]

### 2.3 Reflection / Reflexion

Reflexion 的核心是**三元循环**：

```
Actor → 执行任务
Evaluator → 检查结果质量
Self-Reflector → 产出"经验教训"写回记忆

下次 Actor 执行时，记忆中的教训作为额外上下文
```

**适用场景：** 代码生成/修复、SQL 优化、写作润色

**不适用的场景：** 事实核验——模型自己编造的错误无法通过内部反思自我纠正，必须依赖外部验证锚点 [来源: 2026 大厂 Agent 面试全解析, Baidu Developer]

### 2.4 混合模式（生产环境最常用）

2026 年实际生产中极少纯用一种模式。常见混合架构：

| 架构 | 描述 |
|------|------|
| **Plan → ReAct Execution** | 规划器产出蓝图，每个 step 用 ReAct 灵活执行 |
| **Supervisor → Workers (ReAct)** | Supervisor 路由到专精 Agent，每个 Worker 运行自己的 ReAct 循环 |
| **Reflection wrapping Plan** | 外层 Reflexion 评估规划质量，失败时整轮重做 |
| **Graph → ReAct Leaves** | LangGraph 编排宏观流程，叶子节点用 ReAct 调工具 |

[来源: FutureAGI: Agent Architecture Patterns in 2026 — The Five Named Shapes]

---

## 3. 关键流程 / 算法 / 设计步骤

### 3.1 ReAct 死循环防护

```
Step 1: 硬限制 max_turns = 12-15
Step 2: 重复检测 — 连续 3 次调用同一工具+参数 → 强制退出
Step 3: 无进展检测 — 3 轮无新信息获取 → 打断
Step 4: 超时检测 — 整个任务设最大执行时间
```

[来源: 字节面试题 "Agent调了三个工具就死循环了", 2026]

### 3.2 规划粒度控制

```
太粗: "生成Q1报告" → 无法执行
刚好: "1.拉取CRM 2.拉取ERP 3.Join 4.计算 5.生成PDF"
太细: "1.1打开CRM 1.1.1点击XX菜单 1.1.2输入日期" → Token爆炸
```

**Goldilocks 原则：** 每个 step 是人能独立完成的最小可交付单元。

[来源: kamacoder.com ReAct/Reflection/Planning 面试专题]

### 3.3 生产数据基准

来自 Baidu Developer 2026 文章的真实数据：
- Plan-and-Execute vs 纯 ReAct：**+27% 任务完成率，−42% 工具调用数，−35% 执行时间**
- Agent Genome 论文（Apr 2026, 347 生产 ReAct 轨迹分析）：Execute→Verify 转换概率仅 **2.1%**——验证严重不足
- 引入运行时治理后：**+6.2% 成功率，−44% Token 消耗**

[来源: "Your Agent Has a Genome" arXiv:2606.15579; Baidu Developer AI Agent Framework Guide]

---

## 4. 实际应用场景

| 场景 | 推荐模式 | 原因 |
|------|---------|------|
| 客户退款处理 | ReAct | 1-3 步，每步依赖前一步结果 |
| 月度财务报告生成 | Plan-and-Execute | 5+ 确定性步骤，需审计 |
| 代码 Bug 修复 | Reflection | 需要 Trial-and-Error |
| 竞品分析 | Hybrid: Supervisor + ReAct Workers | 多源数据，需并行+交叉验证 |
| 知识库问答 | ReAct + RAG 检索 | 多轮检索策略 |

[来源: dev.to "ReAct, Plan-and-Execute, or Reflection?", 2026]

---

## 5. 常见误区

1. **"用 ReAct 就够，不需要 Plan-and-Execute"**——5 步以上的确定性任务，Plan-and-Execute 可降低 42% 工具调用和 35% 执行时间 [来源: Baidu Developer]
2. **"Reflection 能自我修复幻觉"**——不能。模型无法内部验证自己编造的事实，必须外部锚点 [来源: 2026 大厂 Agent 面试全解析]
3. **"Thought 是模型免费赠送的"**——不是。每个 Thought 都消耗 Token 和延迟。生产 Agent 应使用 reasoning-native 模型的内置推理能力，避免双重消耗 [来源: FutureAGI 2026 Architecture Guide]
4. **"Planner 和 Executor 用同一个模型就行"**——可以但不经济。混合方案（强模型规划 + 便宜模型执行）在 >3 步任务上总成本更低 [来源: Oracle Integration Blog]

---

## 6. 面试关注点 / 工程取舍

- **ReAct vs Plan-and-Execute 选型：** 如果能提前列出所有工具→Plan-and-Execute；如果不知道中间结果就无法列出→ReAct
- **死循环防护：** 硬限制 + 重复检测 + 无进展检测 + 超时检测
- **Token 成本：** ReAct 每步线性增长；混合方案 1×强模型 + N×便宜模型
- **可审计性：** Plan-and-Execute 天然可审计（蓝图在执行前可人工审）；ReAct 只能追溯
- **面试万能公式：** "场景分析 → 方案选择（对比 trade-off）→ 落地细节 → 踩过的坑 → 效果量化"

[来源: Kamacoder Agent Loop 面试专题; 2026 大厂 Agent 面试通关手册]

---

## 来源

**Survey 论文：**
- **Agentic Design Patterns: A System-Theoretic Framework** (Dao et al., arXiv 2601.19752, Jan 2026) — 12 种 Agentic 设计模式（含 ReAct、Plan-Execute、Reflection 等）的系统论分析，5 个子系统分类。[https://arxiv.org/abs/2601.19752]
- **AI Agent Systems: Architectures, Applications, and Evaluation** (Xu, arXiv 2601.01743, Jan 2026) — 涵盖 planning & control（从 reactive 到 hierarchical multi-step planners）的全面综述。[https://arxiv.org/abs/2601.01743]
- **Making Sense of AI Agents Hype: Adoption, Architectures, and Takeaways from Practitioners** (Su et al., arXiv 2604.00189, Mar 2026) — 分析 138 场工业 talk 的 Agent 架构模式选择实践。[https://arxiv.org/abs/2604.00189]

**其他来源：**
- [ReAct: Synergizing Reasoning and Acting in Language Models (Yao et al., 2022)](https://arxiv.org/abs/2210.03629)
- [Plan-and-Solve Prompting (Wang et al., 2023)](https://arxiv.org/abs/2305.04091)
- [Reflexion: Language Agents with Verbal Reinforcement Learning (Shinn et al., 2023)](https://arxiv.org/abs/2303.11366)
- [Agent Architecture Patterns in 2026: The Five Named Shapes](https://futureagi.com/blog/agent-architecture-patterns-2026/)
- [ReAct, Plan-and-Execute, or Reflection? (dev.to, 2026)](https://dev.to/gabrielanhaia/react-plan-and-execute-or-reflection-the-three-agent-patterns-every-engineer-needs-in-2026-355p)
- [ReAct vs Plan & Execute (Oracle Integration Blog, 2026)](https://blogs.oracle.com/integration/react-vs-plan-execute-choosing-the-right-agent-thinking-pattern-in-oracle-integration)
- [Agent Genome: Sequence-Level Behavioral Analysis (arXiv:2606.15579, Apr 2026)](https://arxiv.org/abs/2606.15579)
- [AI Agent框架设计全解析 (Baidu Developer, 2026)](https://developer.baidu.com/article/detail.html?id=6794083)
- [ReAct、Reflection、规划执行：面试专题 (kamacoder.com)](https://notes.kamacoder.com/llm/app/react_reflection_planning.html)
- [2026 Agent面试终极攻略 (CSDN)](https://blog.csdn.net/2301_80239908/article/details/161602094)
