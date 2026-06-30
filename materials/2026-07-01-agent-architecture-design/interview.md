# Agent Architecture Design — Mock Interview

面试题设计依据：系统设计题型来自当前 Agent 工程实践的公开文档能力边界，包括 Anthropic workflow/agent patterns、OpenAI Agents SDK、LangGraph、Qwen-Agent、Coze、AutoGen，以及 OWASP 对 LLM 应用安全风险的分类。未使用未验证的私下面经原文。[来源: Anthropic Building Effective Agents][来源: OpenAI Agents SDK][来源: LangGraph Overview][来源: Qwen-Agent][来源: Coze Docs][来源: Microsoft AutoGen][来源: OWASP LLM Top 10]

## 1. 架构总览

你要从 0 到 1 设计一个企业内部知识助手，它可以查文档、查工单、调用内部系统创建任务。请给出整体架构。

题型依据：RAG、工具调用、guardrails、tracing 和 production runtime 是 OpenAI Agents SDK、LangGraph、Qwen-Agent 等官方文档共同覆盖的 Agent 系统能力。[来源: OpenAI Agents SDK][来源: LangGraph Overview][来源: Qwen-Agent]

追问：

- 哪些部分是 workflow，哪些部分是 agent？
- 工具层怎么设计？
- 如何做权限隔离？
- 如何避免 prompt injection？

## 2. Workflow vs Agent

什么时候你会拒绝使用 Agent，而选择固定 workflow？请结合真实业务场景说明。

题型依据：Anthropic 明确区分 workflows 和 agents，并建议在复杂度收益明确时再采用 agentic systems。[来源: Anthropic Building Effective Agents]

追问：

- 如果业务方坚持“要更智能”，你如何解释 trade-off？
- 什么时候可以从 workflow 演进到 agent？

## 3. 工具调用与副作用

设计一个 Agent 帮销售自动更新 CRM、生成邮件、安排会议。哪些操作可以自动执行，哪些需要用户确认？

题型依据：OWASP 对 excessive agency、敏感信息泄露和不安全工具/插件设计的风险分类。[来源: OWASP LLM Top 10]

追问：

- 如何保证工具幂等？
- 如何处理工具超时和部分成功？
- 如何审计一次工具调用链？

## 4. 状态与记忆

一个长任务 Agent 需要连续运行 30 分钟，中间可能等待用户确认，也可能工具失败重试。你如何设计状态管理？

题型依据：LangGraph 对 durable execution、human-in-the-loop 和 memory 的定位。[来源: LangGraph Overview]

追问：

- 状态存在数据库、Redis 还是向量库？
- 长期记忆和 execution state 有什么区别？
- 如何恢复中断任务？

## 5. 框架选型

现在有 LangGraph、OpenAI Agents SDK、Qwen-Agent、Coze、AutoGen。你会如何为一个面向国内企业客户的 Agent 产品选型？

题型依据：各框架/平台官方文档中公开的能力边界和生态定位。[来源: LangGraph Overview][来源: OpenAI Agents SDK][来源: Qwen-Agent][来源: Coze Docs][来源: Microsoft AutoGen]

追问：

- 如果必须私有化部署呢？
- 如果目标是快速验证业务呢？
- 如果目标是复杂长任务可靠执行呢？

## 6. 评测与线上问题定位

你的 Agent 上线后用户投诉“经常答非所问，有时乱调用工具”。你会怎么建立评测和观测体系？

题型依据：OpenAI Agents SDK 的 tracing/guardrails，LangGraph 的 production runtime，以及 OWASP 风险分类。[来源: OpenAI Agents SDK][来源: LangGraph Overview][来源: OWASP LLM Top 10]

追问：

- 需要哪些离线测试集？
- trace 里必须记录什么？
- 如何做回归测试？

## 7. 成本与延迟

一个 Agent 平均要调用 5 次模型、3 次检索、2 次工具，P95 延迟超过 20 秒。你如何优化？

题型依据：Anthropic 关于优先保持简单和 workflow patterns 的建议，结合 Agent runtime 的 trace/cost/latency 观测需求。[来源: Anthropic Building Effective Agents][来源: OpenAI Agents SDK]

追问：

- 哪些步骤可以并行？
- 如何做模型路由？
- 如何设置 budget 和 early stop？

## 8. Multi-Agent 取舍

业务方提出“我们做一个多个专家 Agent 协作的系统，一个负责检索，一个负责规划，一个负责执行，一个负责评审”。你如何评价这个方案？

题型依据：Anthropic 的 orchestrator-workers pattern 和 AutoGen 的 multi-agent framework 定位。[来源: Anthropic Building Effective Agents][来源: Microsoft AutoGen]

追问：

- multi-agent 的收益是什么？
- 风险是什么？
- 什么情况下你会真的采用？

## 来源

- Anthropic, Building effective agents: https://www.anthropic.com/engineering/building-effective-agents
- OpenAI Agents SDK documentation: https://openai.github.io/openai-agents-python/
- LangGraph documentation: https://docs.langchain.com/oss/python/langgraph/overview
- Microsoft AutoGen documentation: https://microsoft.github.io/autogen/
- Qwen-Agent GitHub: https://github.com/QwenLM/Qwen-Agent
- Coze documentation: https://www.coze.cn/open/docs/guides/welcome
