# Agent Architecture Design

## 1. 定义

Agent Architecture Design 讨论的不是“怎么写一个会调用工具的 prompt”，而是怎么把 LLM、工具、状态、知识、策略、安全、评测和运行时组织成一个可上线、可调试、可扩展的系统。

本教案的主要归纳综合自 Anthropic 对 workflows/agents 的划分、OpenAI Agents SDK 的 agents/tools/handoffs/guardrails/tracing 抽象、LangGraph 的 durable execution/human-in-the-loop/memory 文档、Qwen-Agent 的 function calling/MCP/RAG/code interpreter 能力，以及 OWASP 对 LLM 应用风险的分类。[来源: Anthropic Building Effective Agents][来源: OpenAI Agents SDK][来源: LangGraph Overview][来源: Qwen-Agent][来源: OWASP LLM Top 10]

一个工程化 Agent 通常包含 8 层：

1. **入口层**：接收用户请求、会话身份、权限、租户、上下文。
2. **意图与路由层**：判断请求适合固定 workflow、单 Agent、RAG、代码执行、多 Agent，还是直接回答。
3. **上下文层**：管理短期上下文、长期记忆、检索结果、任务状态、预算。
4. **模型层**：选择模型、构造消息、处理结构化输出、降级与重试。
5. **规划层**：决定下一步行动，例如 ReAct、Plan-Execute、Reflection、状态机。
6. **工具层**：定义工具 schema、权限、幂等性、超时、沙箱和审计。
7. **运行时层**：负责 durable execution、并发、队列、断点恢复、人类确认。
8. **观测与评测层**：记录 trace、指标、失败样本，做离线评测和线上监控。

## 2. Workflow vs Agent

Anthropic 对这件事的划分很实用：

- **Workflow**：执行路径由开发者预先写死或半写死，例如 prompt chaining、routing、parallelization、orchestrator-workers、evaluator-optimizer。
- **Agent**：LLM 能根据环境反馈动态决定下一步，工具调用链不完全预设。

这个划分来自 Anthropic 的 "Building effective agents"，其中 workflow 被定义为通过预定义代码路径编排 LLM 和工具，agent 则由 LLM 动态指导自身流程和工具使用。[来源: Anthropic Building Effective Agents]

面试里最容易犯的错是把所有东西都叫 Agent。一个生产系统往往是 **workflow 外壳 + agent 内核**：外层用状态机/工作流保证边界和可控性，内层在局部任务里允许模型自主规划。

判断标准：

| 任务特征 | 更适合 |
|---|---|
| 步骤固定、失败代价高、合规要求强 | Workflow |
| 路径不确定、需要探索、工具反馈决定下一步 | Agent |
| 多个子任务可并发处理 | Orchestrator-workers / Multi-agent |
| 需要自动检查和修正 | Evaluator-optimizer / Reflection |

## 3. 核心架构图

```text
User/API
  |
  v
Request Gateway
  - auth / tenant / quota / input filtering
  |
  v
Task Router
  - direct answer
  - RAG workflow
  - single-agent loop
  - code/browser agent
  - human escalation
  |
  v
Agent Runtime
  - state machine / graph
  - planner
  - memory manager
  - tool executor
  - budget manager
  - guardrails
  |
  +--> Model Provider(s)
  +--> Retrieval / Memory Store
  +--> Tool Registry / MCP Servers
  +--> Sandbox / Browser / Code Runner
  +--> Human Approval
  |
  v
Trace + Eval + Metrics
```

这个图的关键点：**Agent 不应该直接裸连数据库、支付、生产 API 或用户私有数据**。中间必须有工具执行层，处理权限、参数校验、幂等、审计和回滚。

## 4. 几种主流实现范式

### 4.1 状态机 / Graph

LangGraph 的核心价值是把 Agent 表达成有状态图：节点代表模型调用、工具调用、人工审批或业务逻辑；边代表状态转移。它强调 durable execution、human-in-the-loop、memory 和 production deployment。

这一节依据 LangGraph 官方文档对 durable execution、human-in-the-loop、memory、deployment 的定位总结。[来源: LangGraph Overview]

适合：

- 需要中断恢复的长任务。
- 要求每一步可追踪、可审计。
- 需要人工确认后继续。
- 业务流程复杂，不能只靠 while loop。

### 4.2 SDK / Runtime

OpenAI Agents SDK 把 agents、handoffs、guardrails、sessions、tracing 做成一套运行时抽象。它适合快速组织单 Agent 或多 Agent 应用，重点是工具调用、handoff 和 trace。

这一节依据 OpenAI Agents SDK 官方文档中的 agents、tools、handoffs、guardrails、sessions 和 tracing 模块总结。[来源: OpenAI Agents SDK]

适合：

- 使用 OpenAI 生态。
- 需要标准化 tracing。
- 多个 Agent 之间有清晰职责转交。

### 4.3 框架型 Agent

Qwen-Agent 提供 function calling、MCP、code interpreter、RAG、Chrome extension 等能力，定位是围绕 Qwen 模型构建应用的 agent framework。

这一节依据 Qwen-Agent GitHub README 对 Function Calling、MCP、Code Interpreter、RAG 和 browser assistant 能力的说明总结。[来源: Qwen-Agent]

适合：

- 面向阿里/Qwen 生态。
- 需要把 function calling、RAG、代码执行、MCP 串起来。
- 需要中文模型和国内云生态兼容。

### 4.4 平台型 Agent Builder

Coze/扣子这类平台把 bot、workflow、knowledge、plugin、database、publish channels 做成低代码/平台能力。它反映了大厂产品化 Agent 的常见拆法：知识库、插件、工作流、变量、发布渠道和运营后台分离。

这一节依据 Coze 官方开放文档对 bot、workflow、knowledge、plugin 等平台模块的说明总结。[来源: Coze Docs]

适合：

- 业务团队快速搭建。
- 多渠道发布。
- 需要内置知识库和插件市场。

### 4.5 Multi-Agent Framework

AutoGen/CrewAI 这类框架强调多个 agent 协作、角色分工、对话式协同和任务编排。面试中要注意：multi-agent 很容易带来成本上升、错误传播、调试困难和一致性问题，所以不要默认作为第一选择。

AutoGen 的多 agent 协作定位来自 Microsoft AutoGen 官方文档；关于成本、错误传播和调试复杂度是结合 Anthropic “保持简单、仅在收益明确时增加 agentic complexity”的建议推导。[来源: Microsoft AutoGen][来源: Anthropic Building Effective Agents]

## 5. 设计一个 Agent 的步骤

### Step 1: 明确任务边界

先回答：

- 用户目标是什么？
- 任务是否有确定流程？
- Agent 能不能执行有副作用操作？
- 错误代价是什么？
- 需要哪些外部系统？
- 哪些步骤必须由人确认？

如果这些问题答不清楚，后面的模型选型和框架选型都没有意义。

### Step 2: 选择控制流

常见控制流：

```text
fixed workflow:
  input -> classify -> retrieve -> answer -> cite

ReAct loop:
  observe -> think -> act(tool) -> observe -> ... -> final

plan-execute:
  make plan -> execute step -> update plan -> final

graph:
  node A -> conditional edge -> node B/C -> human approval -> node D
```

工程上推荐优先级：

1. 能用固定 workflow，不用 Agent。
2. 能用单 Agent，不用 multi-agent。
3. 能用有限状态图，不用无限自主循环。
4. 所有循环都必须有停止条件、预算和失败路径。

### Step 3: 设计工具层

工具不是普通函数，它是 Agent 的行动边界。每个工具要定义：

- **schema**：参数类型、必填字段、枚举、范围。
- **permission**：谁能调用，什么场景能调用。
- **idempotency**：重复调用会不会造成重复扣款、重复发消息。
- **side effect**：是否修改外部状态。
- **timeout/retry**：超时和重试策略。
- **audit**：记录调用者、参数摘要、结果、错误。
- **confirmation**：高风险操作是否需要人工确认。

高风险工具要拆成两段：

```text
prepare_action -> human_confirm -> commit_action
```

### Step 4: 设计上下文与记忆

Agent 的上下文通常分成：

- **working context**：当前轮对话、工具观测、短期任务状态。
- **retrieved context**：从知识库/文档/数据库检索出的内容。
- **long-term memory**：用户偏好、历史任务、项目事实。
- **execution state**：已经执行到哪一步、预算用了多少、失败过什么。

上下文设计的核心是选择和压缩，而不是“都塞进窗口”。长上下文模型降低了上下文管理压力，但没有消除噪声、成本和安全问题。

### Step 5: 加运行时护栏

必要护栏：

- 最大步骤数。
- 最大 token/cost。
- 工具调用白名单。
- 高风险工具人工确认。
- 输出结构校验。
- prompt injection 检测与隔离。
- 检索内容和用户指令分层。
- 失败降级：转人工、固定模板回答、只读模式。

### Step 6: 建评测和观测

上线前至少要有：

- golden tasks：固定任务集。
- tool-call accuracy：工具选择和参数是否正确。
- task success rate：最终任务是否完成。
- latency/cost：平均和 P95。
- safety eval：越权、泄露、prompt injection。
- trace：每一步模型输入输出、工具调用、状态转移、错误。

没有 trace 的 Agent 系统不可维护。你无法只看最终回答来判断错在 prompt、检索、工具、模型、状态机还是权限。

工具权限、越权调用、prompt injection 和 excessive agency 的风险归纳参考 OWASP Top 10 for LLM Applications；trace/eval 的运行时抽象参考 OpenAI Agents SDK 和 LangGraph 文档。[来源: OWASP LLM Top 10][来源: OpenAI Agents SDK][来源: LangGraph Overview]

## 6. 框架选型

| 选择 | 优点 | 风险 | 适合场景 |
|---|---|---|---|
| 手写 workflow | 可控、依赖少、易审计 | 灵活性弱 | 流程固定的业务 |
| LangGraph | 状态清晰、可恢复、HITL 友好 | 学习成本 | 复杂长任务 |
| OpenAI Agents SDK | 工具、handoff、trace 集成好 | 生态绑定 | OpenAI-first 项目 |
| Qwen-Agent | 支持 Qwen、MCP、RAG、Code Interpreter | 生态边界 | 国内模型/阿里生态 |
| Coze/扣子 | 平台化快、业务友好 | 深度定制受限 | 快速业务验证 |
| AutoGen/CrewAI | 多 Agent 协作抽象强 | 成本和调试复杂 | 研究/复杂协作探索 |

面试回答的高级点：**框架不是架构**。框架只解决部分抽象，真正的架构还要定义数据边界、权限边界、运行时可靠性、评测闭环和成本控制。

## 7. 常见误区

1. **把 prompt 当架构**：prompt 只是模型层输入，不负责权限、状态和恢复。
2. **过早 multi-agent**：多个 Agent 不自动更聪明，通常先更贵、更慢、更难 debug。
3. **工具裸奔**：让模型直接决定调用高风险工具，没有权限和确认。
4. **没有失败路径**：Agent 卡循环、工具超时、检索无结果时没有兜底。
5. **只做 demo，不做 eval**：能跑一次不代表能上线。
6. **把 RAG 当万能记忆**：RAG 是检索，不等于长期记忆和执行状态。

## 8. 大厂/AI 独角兽面试关注点

阿里/腾讯/字节/DeepSeek/智谱/MiniMax 这类公司的 Agent 方向，常见追问不是“你知道 LangChain 吗”，而是：

- 给你一个业务场景，怎么判断该用 workflow、single-agent 还是 multi-agent？
- 工具调用失败或参数错了，怎么发现和恢复？
- 怎么防 prompt injection 导致越权调用？
- 怎么设计 memory，不污染上下文？
- 怎么压成本和延迟？
- 怎么做离线评测和线上观测？
- 你如何证明这个 Agent 比固定流程更好？

## 9. 一句话总结

Agent Architecture Design 的核心是：**用工程化控制面包住模型的不确定性，让 Agent 在受控边界里自主行动**。

## 来源

- Anthropic, Building effective agents: https://www.anthropic.com/engineering/building-effective-agents
- OpenAI Agents SDK documentation: https://openai.github.io/openai-agents-python/
- LangGraph documentation: https://docs.langchain.com/oss/python/langgraph/overview
- Microsoft AutoGen documentation: https://microsoft.github.io/autogen/
- Qwen-Agent GitHub: https://github.com/QwenLM/Qwen-Agent
- Coze documentation: https://www.coze.cn/open/docs/guides/welcome
- OWASP Top 10 for LLM Applications: https://owasp.org/www-project-top-10-for-large-language-model-applications/
