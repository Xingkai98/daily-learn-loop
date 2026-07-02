# Agent Planning & Reasoning — Exam

> 题目设计依据：综合自字节/阿里/腾讯 2025-2026 Agent 面试真题、InfoQ Agent Loop 专题、kamacoder 面经整理、Oracle ReAct vs Plan-Execute 实战指南。

---

## 1. Agent 的三种核心推理模式 (⭐)

选择题

以下哪一项正确描述了 ReAct、Plan-and-Execute 和 Reflection 三种 Agent 模式的核心区别？

A. ReAct 是先规划再行动，Plan-and-Execute 是边行动边思考，Reflection 是只行动不思考  
B. ReAct 是 Thought → Action → Observation 循环；Plan-and-Execute 是先由 Planner 生成步骤再由 Executor 执行；Reflection 是执行后由 Evaluator 评估并反馈教训  
C. 三种模式本质相同，只是不同框架的不同叫法  
D. Reflection 是 ReAct 的子集，专门处理代码生成场景  

> 来源：FutureAGI ReAct Glossary & InfoQ Agent Loop 专题

---

## 2. ReAct 循环的组成部分 (⭐)

选择题

ReAct 模式的标准循环中包含以下哪几个环节？

A. Plan → Execute → Review → Deploy  
B. Thought → Action → Observation → Thought → ...  
C. Query → Retrieve → Rank → Generate  
D. Listen → Understand → Respond → Learn  

> 来源：Yao et al. ReAct 论文 (2022)

---

## 3. ReAct 的死循环问题 (⭐⭐)

简答题

字节面试真题："你的 Agent 调了三个工具就死循环了，异常处理在哪写的？"

请回答以下三个方面：
1. ReAct 为什么会陷入死循环？举一个具体场景。
2. 设计至少三种工程防护措施。
3. 防护措施各自的代价是什么？

> 来源：华为云博客 字节 Agent 面试题 (2026)

---

## 4. Plan-and-Execute 的核心设计决策 (⭐⭐)

简答题

在设计 Plan-and-Execute Agent 时，Planner 产出的步骤粒度是一个核心决策。

1. 步骤过粗（如"生成 Q1 报告"）和过细（如"1.1.1 打开菜单 → 1.1.2 点击按钮"）分别会导致什么问题？
2. 如何确定合适的步骤粒度？给出你的判断标准。
3. 如果执行过程中某个 step 失败了，重规划机制应该怎样设计？

> 来源：InfoQ Plan-and-Execute 专题; Oracle ReAct vs Plan-Execute Guide

---

## 5. Reflection 的适用边界 (⭐⭐)

简答题

Reflection/Reflexion 模式可以让 Agent "自我反思"。

1. 这个模式适合什么类型的任务？不适合什么类型？
2. 为什么 Reflection 不适合事实核验类任务？
3. 如何在实际项目中判断 Agent 的反思是"真的有效"还是"自我安慰"？

> 来源：2026 大厂 Agent 面试全解析 (Baidu Developer); Reflexion 论文 Shinn et al. 2023

---

## 6. 三种模式的选型决策 (⭐⭐⭐)

场景题

你所在团队要为以下三个业务场景选择 Agent 推理模式。请分别对每个场景：
- 选出最合适的模式（ReAct / Plan-and-Execute / Reflection / Hybrid）
- 解释选择理由，说明放弃了什么（Trade-off）
- 描述关键工程防护措施

**场景 A：** 用户说"帮我查一下上周下的那个订单发货没，如果没发就取消重新下一单"（涉及 2-5 步工具调用）

**场景 B：** 每月 1 号自动从 CRM、ERP、财务系统拉取数据，生成月度经营分析报告（8-15 步骤，需合规审计）

**场景 C：** 用户提交了一段 Python 代码，要求 Agent 找出 bug 并修复，修复后的代码要通过单元测试（需要 Trial-and-Error 迭代）

> 来源：kamacoder Agent Loop 面试专题; FutureAGI 2026 Architecture Patterns Survey

---

## 来源

- [Yao et al., ReAct: Synergizing Reasoning and Acting in Language Models (2022)](https://arxiv.org/abs/2210.03629)
- [Wang et al., Plan-and-Solve Prompting (2023)](https://arxiv.org/abs/2305.04091)
- [Shinn et al., Reflexion (2023)](https://arxiv.org/abs/2303.11366)
- [字节面试官：Agent 调了三个工具就死循环了 (华为云博客, 2026)](https://bbs.huaweicloud.com/blogs/477120)
- [InfoQ: Plan-and-Execute 专题 (2026)](https://xie.infoq.cn/article/159ae42cc328312c526a49cb7)
- [kamacoder: ReAct/Reflection/规划执行面试专题](https://notes.kamacoder.com/llm/app/react_reflection_planning.html)
- [2026大厂 Agent 面试全解析 (Baidu Developer)](https://developer.baidu.com/article/detail.html?id=7670839)
- [FutureAGI: Agent Architecture Patterns 2026](https://futureagi.com/blog/agent-architecture-patterns-2026/)
