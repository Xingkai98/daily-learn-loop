# Agent Architecture Design — Exam Answers

答案依据：本答案综合 Anthropic、OpenAI Agents SDK、LangGraph、Qwen-Agent、Coze、AutoGen 与 OWASP LLM Top 10。每题后的来源标注说明主要依据。[来源: Anthropic Building Effective Agents][来源: OpenAI Agents SDK][来源: LangGraph Overview][来源: Qwen-Agent][来源: Coze Docs][来源: Microsoft AutoGen][来源: OWASP LLM Top 10]

## 1. Workflow vs Agent（⭐）

Workflow 的路径主要由开发者定义，Agent 的路径由模型根据反馈动态决定。

更适合 workflow：

- 退款审批、报销提交、合规审核等步骤固定且副作用强的流程。
- RAG 问答：分类、检索、重排、生成、引用，流程比较稳定。

更适合 agent：

- 代码修复：需要读文件、运行测试、根据错误继续探索。
- 复杂研究任务：需要不断搜索、比较来源、调整查询。

关键不是名字，而是控制权在谁手里。生产系统常用 workflow 外壳限制边界，在局部任务中允许 Agent 自主规划。

主要依据：Anthropic 对 workflow/agent 的定义和“从简单方案开始”的建议。[来源: Anthropic Building Effective Agents]

## 2. 架构分层（⭐）

合格答案应覆盖至少 5 层：

- 入口层：身份、租户、权限、quota、输入过滤。
- 路由层：判断 direct answer、RAG、workflow、agent、human escalation。
- 上下文层：管理历史、检索、记忆、执行状态、预算。
- 模型层：模型选择、prompt/messages、结构化输出、重试降级。
- 规划层：ReAct、Plan-Execute、Reflection、状态图。
- 工具层：schema、权限、沙箱、幂等、审计。
- 运行时层：durable execution、队列、并发、人工确认。
- 观测评测层：trace、指标、离线 eval、线上监控。

主要依据：OpenAI Agents SDK 的 tracing/guardrails/sessions 抽象、LangGraph 的 durable execution/memory/HITL，以及 OWASP 的安全风险分类。[来源: OpenAI Agents SDK][来源: LangGraph Overview][来源: OWASP LLM Top 10]

## 3. 工具调用安全（⭐⭐）

可以自动执行：

- OCR/解析发票。
- 读取用户已授权的报销规则。
- 草拟报销单。
- 校验字段格式。

必须 human-in-the-loop：

- 最终提交审批。
- 修改金额、收款账户、成本中心等高风险字段。
- 发现票据异常时继续提交。
- 代表用户发送正式说明。

原因：提交报销有业务副作用和合规责任。正确架构应拆成 prepare/preview/confirm/commit，而不是让模型直接提交。

主要依据：OWASP 对 excessive agency 和工具/插件越权风险的分类；human-in-the-loop 运行时依据 LangGraph 文档。[来源: OWASP LLM Top 10][来源: LangGraph Overview]

## 4. 控制流选择（⭐⭐）

推荐混合架构：

```text
input
  -> intent router
  -> order query workflow
  -> policy explanation RAG
  -> refund eligibility rules + LLM explanation
  -> refund prepare
  -> human/user confirmation
  -> refund commit tool
  -> exception -> human support
```

查询订单和发起退款应是 workflow；解释政策可用 RAG；异常分析可用 single-agent；不需要 multi-agent。退款提交必须有确认和审计。

主要依据：Anthropic 的 routing/workflow patterns；高风险工具确认依据 OWASP LLM Top 10。[来源: Anthropic Building Effective Agents][来源: OWASP LLM Top 10]

## 5. 上下文与记忆（⭐⭐）

- working context：当前任务的对话、工具结果、临时观察。
- retrieved context：从知识库检索出来的文档片段。
- long-term memory：用户偏好、长期事实、历史项目背景。
- execution state：当前任务执行到哪一步、已经调用过什么、失败次数、预算。

不能全塞上下文，因为会增加成本和延迟，引入噪声，扩大 prompt injection 面，污染模型判断，并让历史错误持续影响输出。

主要依据：LangGraph 对 memory/state 的区分，以及 OWASP 对 prompt injection 和敏感信息泄露风险的分类。[来源: LangGraph Overview][来源: OWASP LLM Top 10]

## 6. 可观测性与评测（⭐⭐⭐）

先看：

- task success rate 分业务类型的变化。
- tool-call accuracy：选错工具还是参数错。
- retrieval hit rate / rerank quality。
- model latency、tool latency、timeout、retry。
- permission denied、schema validation error。
- trace 中每一步模型输入输出和状态转移。

定位方法：

- prompt 问题：同样检索和工具结果下，模型决策变差。
- 检索问题：相关文档未召回或召回噪声变多。
- 工具问题：工具错误率、超时、schema mismatch 上升。
- 模型问题：某个模型版本上线后失败集中出现。
- 权限问题：permission denied 或租户/角色相关失败上升。

主要依据：OpenAI Agents SDK tracing、LangGraph production runtime，以及 OWASP 安全风险分类。[来源: OpenAI Agents SDK][来源: LangGraph Overview][来源: OWASP LLM Top 10]

## 7. 框架选型（⭐⭐⭐）

- LangGraph：适合复杂状态图、长任务、human-in-the-loop、断点恢复。
- OpenAI Agents SDK：适合 OpenAI-first、需要 handoff/guardrails/tracing 的系统。
- Qwen-Agent：适合 Qwen/阿里生态，function calling、MCP、RAG、code interpreter 集成。
- Coze/扣子：适合业务快速搭建、知识库、插件、工作流、多渠道发布。
- AutoGen：适合多 Agent 协作探索和研究型任务。

生产系统选型不应只看框架热度。要看模型生态、私有化要求、审计要求、团队能力、可观测性、成本、是否需要长任务恢复。

主要依据：各框架官方文档公布的能力边界。[来源: LangGraph Overview][来源: OpenAI Agents SDK][来源: Qwen-Agent][来源: Coze Docs][来源: Microsoft AutoGen]

## 8. 反模式识别（⭐⭐⭐）

风险：

- 模型能调用任意内部 API，权限边界缺失。
- 没有工具 schema 和参数校验。
- 没有 human-in-the-loop，高风险操作可能直接执行。
- 没有幂等和重试策略，可能重复执行副作用。
- 没有审计和 trace，出错无法追责。
- 没有 prompt injection 防护，外部文本可能诱导越权。
- 没有预算/步骤限制，可能死循环或成本失控。

改进：

```text
用户输入
  -> gateway(auth/quota/input filter)
  -> router
  -> agent runtime(state/budget/guardrails)
  -> tool gateway(schema/permission/rate limit/audit)
  -> low-risk tools auto execute
  -> high-risk tools prepare + human confirm + commit
  -> trace/eval/metrics
```

主要依据：OWASP LLM Top 10 的安全风险、OpenAI Agents SDK 的 guardrails/tracing、LangGraph 的 durable execution/HITL。[来源: OWASP LLM Top 10][来源: OpenAI Agents SDK][来源: LangGraph Overview]

## 来源

- Anthropic, Building effective agents: https://www.anthropic.com/engineering/building-effective-agents
- OpenAI Agents SDK documentation: https://openai.github.io/openai-agents-python/
- LangGraph documentation: https://docs.langchain.com/oss/python/langgraph/overview
- Microsoft AutoGen documentation: https://microsoft.github.io/autogen/
- Qwen-Agent GitHub: https://github.com/QwenLM/Qwen-Agent
- Coze documentation: https://www.coze.cn/open/docs/guides/welcome
- OWASP Top 10 for LLM Applications: https://owasp.org/www-project-top-10-for-large-language-model-applications/
