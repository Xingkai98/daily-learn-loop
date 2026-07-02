# Agent Planning & Reasoning — Mock Interview

> 面试题设计依据：综合字节/阿里/腾讯/百度 2025-2026 Agent 开发岗面试真题、InfoQ Agent Loop 专题、kamacoder 面经、Oracle & FutureAGI 实战指南。

---

## 1. ReAct 基础 (L1→L2)

请描述 ReAct 的完整执行循环。用一个具体的任务（比如帮用户退订单）演示 Thought → Action → Observation 的全过程。每个环节是模型生成的还是工程代码决定的？

追问：
- ReAct 和简单的 Chain-of-Thought 有什么区别？CoT 能做 Agent 吗？
- 如果某一步的 Observation 返回了完全出乎意料的结果（比如查订单返回了 500 错误），ReAct 有什么机制来应对？够不够？
- 用 ReAct 做一个 15 步的复杂任务时，最可能在哪一步出问题？为什么？

> 来源：字节跳动 Agent 面试真题; kamacoder Agent Loop 专题

---

## 2. Plan-and-Execute (L2→L3)

Plan-and-Execute 把"规划"和"执行"拆成两个独立的角色。这个设计解决了什么问题，又引入了什么新问题？

追问：
- Planner 需要多强的模型？Executor 呢？为什么通常用不同规格的模型？
- 执行到第 4 步时发现第 2 步的结果有问题，但第 3 步已经基于它执行完了。你的重规划机制怎么处理？
- Plan-and-Execute 的最大缺陷是什么？在什么场景下它完全不应该被使用？

> 来源：阿里巴巴 Agent 架构设计面试真题; InfoQ Plan-and-Execute 专题

---

## 3. 模式选型 (L3)

你现在有三个任务。请分别为它们选型，讲出决策逻辑和 Trade-off：
- **任务 A：** 用户说"我上周买了三条裤子，有一条尺码不对，帮我换一下"
- **任务 B：** "生成本季度的销售分析报告，数据从 5 个不同系统中拉取"
- **任务 C：** "这段代码有个偶发的 race condition bug，帮我找出来并修复"

追问：
- 任务 A 如果用 Plan-and-Execute 会有什么问题？
- 任务 B 如果用纯 ReAct 呢？成本的账怎么算？
- 在实际团队中，你怎么说服同事接受你的选型？

> 来源：腾讯/阿里混合型 Agent 面试真题; FutureAGI 2026 Architecture Survey

---

## 4. Reflection 的信任问题 (L2→L3)

Reflection/Reflexion 模式让 Agent "自己评价自己"。面试官最喜欢追问：你怎么知道 Agent 的自我反思是真的进步了，而不是在自我安慰？

追问：
- 如果 Agent 的第一次尝试就错了（比如把金额 540000 分写成了 "$54.00"），Reflection 能发现吗？什么情况下能、什么情况下不能？
- 你在生产环境中怎么给 Reflection 加上"外部锚点"？举一个具体例子。
- 如果 Critic 反馈本身就是错的（说改 A，其实应该改 B），Agent 会不会越改越坏？怎么防？

> 来源：腾讯 Agent 反思机制面试真题; Reflexion 论文 Shinn et al. 2023

---

## 5. 死循环与异常恢复 (L3→L4)

字节经典面试题："你的 Agent 调了三个工具就死循环了，异常处理在哪写的？"

追问：
- 你提到的 max_turns 设为多少？这个数字怎么定的？太小会误杀正常任务，太大浪费 Token——你怎么做实验确定最优值？
- "连续三次同一工具+参数"的重复检测——但参数每次都细微不同（大小写、格式差异）呢？你怎么定义"语义相同"？
- Agent 卡住了不是因为死循环，而是因为任务本身太难（比如面对一个从未见过的 API error），它不知道下一步该怎么办。这和死循环有什么区别？处理方式有什么不同？

> 来源：字节跳动 Agent 面试真题 (2026); 华为云博客

---

## 6. 架构演进实战 (L4)

"从 0 到 1 设计一个电商售后 Agent，最开始用纯 ReAct，上线后发现三个问题：高并发时段 Token 成本爆炸、复杂多步任务成功率低、出了问题难以追溯。你怎么一步步演进这个架构？"

追问：
- 你引入了 Plan-and-Execute 还是 Supervisor+Workers？为什么选这个而不是另一个？
- 架构改变后，已有的 Prompt 和 Tool Schema 需要怎么改？
- 怎么衡量架构演进的收益？你用什么指标说服老板这次重构值得做？
- 新一代 reasoning 模型（如 Claude 4.7 的 extended thinking、GPT-5）会改变这些架构选择吗？未来 2-3 年趋势是什么？

> 来源：阿里 P7 Agent 架构师面试真题; FutureAGI LLM Agent Architectures 2026

---

## 来源

- [字节/阿里/腾讯 Agent 面试真题合集 (CSDN 2026)](https://blog.csdn.net/2301_80239908/article/details/161602094)
- [kamacoder: ReAct/Reflection/规划执行面试专题](https://notes.kamacoder.com/llm/app/react_reflection_planning.html)
- [InfoQ: Plan-and-Execute 专题 (2026)](https://xie.infoq.cn/article/159ae42cc328312c526a49cb7)
- [FutureAGI: Agent Architecture Patterns 2026](https://futureagi.com/blog/agent-architecture-patterns-2026/)
- [华为云博客: 字节 Agent 死循环面试题](https://bbs.huaweicloud.com/blogs/477120)
- [Oracle: ReAct vs Plan & Execute Guide](https://blogs.oracle.com/integration/react-vs-plan-execute-choosing-the-right-agent-thinking-pattern-in-oracle-integration)
