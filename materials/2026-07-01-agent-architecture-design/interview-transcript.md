# Agent Architecture Design — Interview Transcript

日期：2026-07-01
主题：Agent Architecture Design
状态：模拟面试通过

## 1. 企业内部知识助手架构设计

问题：从 0 到 1 设计一个企业内部知识助手，可以查公司文档、查历史工单、调用内部系统创建任务，并根据用户权限决定能看什么、能做什么。

用户回答摘要：

- 系统分输入层、上下文层、模型层、规划层、执行层、工具层、运行时层。
- 查文档、查工单抽象为工具。
- 创建任务对 Agent 暴露为工具，但内部应由 workflow 承载。
- 创建任务前校验入参，真正创建前需要人工审核。
- 工具失败要返回有意义的结构化错误，引导模型修正。
- 权限分两层：用户能使用哪些工具，以及工具本身是读、写还是高风险操作。

点评：

- 通过。回答抓住了只读工具、写工具、人工审核、参数校验和权限分层。
- 需要补强：整体应描述为混合架构，而不是泛称“整体是 Agent”；权限必须由 gateway、permission service、tool gateway 强制执行，不能只透传给 Agent；企业知识助手要强调 ACL filtering、租户隔离和 trace 脱敏。

追问：用户无权访问研发一部工单时如何拦截？

用户回答摘要：

- 查询工单工具根据用户部门限制查询范围。
- 工具返回给模型的只应是有权限部门的工单。
- 权限不匹配时要返回结构化信息，让 Agent 区分“没查到”和“无权限”。

点评：

- 通过但需修正：权限范围不能由 Agent 传参决定，必须由服务端强制注入和校验。
- 推荐链路：Gateway 解析用户身份和租户，Permission Service 计算 allowed_scope，Tool Gateway 约束查询范围，检索前后都做 ACL filtering。

追问：RAG 检索到的工单文本包含 prompt injection，如何防止 Agent 当指令执行？

用户第一次回答：未想好。

修正后用户回答摘要：

- 识别这是 prompt injection。
- 检索内容不可信，只能作为数据来源，不能覆盖系统提示词。
- 对“忽略之前指令”“拿到最高权限”“调用所有工具”等高危文本做检测，命中后触发人工审核。
- 高危工具仍需人工审批。
- 工具调用前完整校验用户权限和工具权限。
- 记录 trace，区分原始检索文本、模型处理内容和最终输出。

点评：

- 通过。核心防线完整。
- 还可补充：关键词黑名单不够，要配合语义检测/策略分类；trace 中敏感原文要脱敏、访问控制和设置保留周期。

## 2. Workflow vs Agent 取舍

问题：什么时候拒绝使用 Agent，而选择固定 workflow？请结合真实业务场景说明。

用户回答摘要：

- 主要看需求方对执行逻辑确定性的要求，以及操作是否需要重审批或高风险。
- 真实场景：运营商故障诊断。网络配置操作危险，误操作可能造成网络事故。
- 因此更倾向确定性的 workflow。
- Trade-off 是更高自主性会带来事故风险。
- 演进到 Agent 前，需要落实沙箱执行、权限管理，并在线上得到验证。

点评：

- 通过。运营商网络配置是高风险 workflow 场景的好例子。
- 可补强：故障诊断可拆成已知 runbook workflow、只读日志/指标分析 agent、配置修改 workflow + 审批 + 回滚。
- 演进条件应包括 shadow mode、golden tasks、安全指标、审批回滚、审计和低风险节点先 agent 化。

## 3. 工具调用与副作用

问题：设计一个 Agent 帮销售自动更新 CRM、生成邮件、安排会议。哪些操作可以自动执行，哪些需要用户确认？

用户回答摘要：

- 更新 CRM 可以自动执行，因为危险性相对不大。
- 安排会议也可以自动执行，因为可以取消。
- 发邮件必须用户确认，因为涉及正式沟通或对外交流。
- 工具幂等：每次工具调用生成唯一 ID。
- 工具超时：给 Agent 传超时时间；异步任务拿 ID 后轮询。
- 部分成功：工具保存 checkpoint，记录上次阶段执行状态。
- 审计：记录工具入参、出参、模型拿到结果后的输出、用户权限和执行模式。

点评：

- 部分通过，补充后通过。
- 修正点：CRM 和会议不应笼统自动执行。读 CRM、草拟邮件、查询空闲时间、生成候选时间可以自动；修改 deal amount、pipeline stage、owner、发外部邀请、取消会议等需要确认。
- 幂等不应说“工具调用结果无状态”，而是服务端用 idempotency key 保存提交状态。
- 审计应补充 run_id、trace_id、tool_call_id、model/prompt/tool schema version、approval record、retry/error、latency、敏感字段脱敏。

## 4. 状态与记忆

问题：一个长任务 Agent 连续运行 30 分钟，中间可能等待用户确认，也可能工具失败重试。如何设计状态管理？

用户回答摘要：

- Redis 缓存比较合适，因为中间状态主要在连续运行任务中需要。
- 长期记忆是用户画像或偏好，execution state 是任务执行状态。
- 恢复中断任务需要工具幂等，有整体 ID 区分不同工具调用状态。
- 任务需要 checkpoint，每完成一个阶段都保存当前结果。

点评：

- 部分通过，补充后通过。
- 关键修正：长任务 execution state 不建议只放 Redis。Redis 适合锁、队列和短期缓存，权威状态应放数据库或 durable runtime/checkpointer。
- 需要持久化 run_id、current_node、completed_steps、tool_call_status、approval_status、budget_used、error_count 和 checkpoint。
- 恢复时从安全 checkpoint 继续；写工具先用 idempotency key 查询 committed/prepared/failed/unknown 状态，避免重复执行。

## 5. 框架选型

问题：LangGraph、OpenAI Agents SDK、Qwen-Agent、Coze、AutoGen 中，如何为国内企业客户 Agent 产品选型？

用户回答摘要：

- 国内企业倾向 Qwen-Agent，因为国内生态支持好，Qwen 模型和 Agent 工具适配。
- 私有化可考虑开源 Qwen 模型。
- 快速验证业务可以用 Coze 做 workflow，或用 AutoGen 做多 Agent 探索。
- 复杂长任务可靠执行用 LangChain，因为有任务状态持久化和 checkpoint。

点评：

- 通过。
- 术语修正：应说 LangGraph，不是 LangChain。LangGraph 强调 graph、state、checkpoint、durable execution 和 human-in-the-loop。
- Qwen-Agent 私有化应表述为“评估开源 Qwen 模型 + vLLM/Ollama/OpenAI-compatible serving + Qwen-Agent”，不要笼统说所有 Qwen 能力都开源。
- Coze 适合业务 POC，AutoGen 适合 multi-agent 原型，OpenAI Agents SDK 适合 OpenAI-first 项目。

## 6. 评测与线上问题定位

问题：Agent 上线后用户投诉“经常答非所问，有时乱调用工具”，如何建立评测和观测体系？

用户回答摘要：

- 拆成答非所问和乱调用工具两个问题。
- 对答非所问，沿 trace 看从哪一轮对话开始偏离，定位消息来源是工具输出还是模型幻觉。
- 对乱调用工具，检查上下文中是否注入了需要的工具，以及模型是否选错工具。
- 用完整 trace 逐步缩小根因范围。
- 离线测试集：把问题 case 固化，保存 prompt 和工具输出，在相同条件下测试模型输出和工具调用是否准确。
- 回归测试：每次发布跑 daily cases，改动不能造成恶化。
- Trace 记录每轮给模型的上下文、工具输入输出、用户权限、执行模式和环境信息。

点评：

- 通过。trace-driven debugging 和 regression case 思路正确。
- 可补强：离线集应覆盖 golden tasks、tool selection、tool arguments、no-tool、retrieval、permission/security、历史失败样本和 edge cases。
- Trace 应补充 run_id、router decision、model/prompt/tool schema version、retrieval query/top-k/rerank score、guardrail decision、state transition、latency/token/cost/retry/fallback。

## 7. 成本与延迟

问题：一个 Agent 平均调用 5 次模型、3 次检索、2 次工具，P95 延迟超过 20 秒。如何优化？

用户回答摘要：

- 检索可以并行，让 Agent 一次输出多次检索请求并并行执行。
- 模型路由按任务类型区分：复杂理解任务用更强模型，简单总结或条件判断用小模型。
- Budget 主要是 token，上下文达到阈值后自动压缩。
- Early stop：Agent 每隔几轮根据任务目标自检，如果目标达成就停止。

点评：

- 部分通过，补充后通过。
- 优化前应先看 trace，定位 P95 慢在模型、检索、工具、重试、超时还是 loop 轮数。
- 最有效优化通常是减少 Agent loop：固定路径 workflow 化、批量工具、分类/改写/格式化交给小模型或规则。
- 并行不仅是检索，还包括互不依赖的只读工具；写操作和有顺序约束的流程不能随便并行。
- Budget 不只是 token，还包括 max_steps、max_model_calls、max_tool_calls、max_retries、max_wall_time、max_cost、per-tool timeout。

## 8. Multi-Agent 取舍

问题：业务方提出多个专家 Agent 协作，一个负责检索、一个负责规划、一个负责执行、一个负责评审。如何评价？

用户回答摘要：

- 收益：每个 Agent 专注自己的任务，不同任务上下文不互相污染。
- 可按任务选择不同模型，平衡成本。
- 执行和评审分离，降低同一模型自评偏差。
- 风险：不确定性增大，上游 Agent 错误会被下游放大。
- 采用条件：场景天然需要多种任务和协作关系。

点评：

- 通过。
- 可补强的风险：成本和延迟上升、trace/debug 更复杂、多 Agent 目标不一致、中间信息传递失真、权限边界更难管理、最终责任归因困难。
- 采用条件应更严格：任务天然可拆、子任务可并行、不同子任务需要不同工具/权限/模型，并且 eval 证明 multi-agent 比 single-agent 或 graph workflow 成功率更高、成本和延迟可接受。

## 总体评价

优点：

- 能用 workflow vs agent 的边界来分析高风险业务场景。
- 对工具权限、人工审批、幂等、checkpoint、trace、回归测试有工程意识。
- 能结合运营商故障诊断、CRM、工单系统等真实业务场景回答。
- 对国内模型生态、Qwen-Agent、Coze、AutoGen、LangGraph 的定位基本准确。

待加强：

- 权限边界必须由系统侧强制执行，不能依赖 Agent 传参或自觉遵守。
- 高风险写操作要更保守，CRM/会议/任务创建都要细分读、草稿、低风险写、高风险写。
- Execution state 的权威存储应持久化，不应只放 Redis。
- Prompt injection 需要形成固定答题模板：untrusted data、内容检测、tool gateway、HITL、output guardrail、trace/audit。
- 性能优化要先看 trace 找 P95，再减少 loop，不要直接从并行检索开始。

结论：模拟面试通过，可以进入归档阶段。

## 来源

- Anthropic, Building effective agents: https://www.anthropic.com/engineering/building-effective-agents
- OpenAI Agents SDK documentation: https://openai.github.io/openai-agents-python/
- LangGraph documentation: https://docs.langchain.com/oss/python/langgraph/overview
- Qwen-Agent GitHub: https://github.com/QwenLM/Qwen-Agent
- OWASP Top 10 for LLM Applications: https://owasp.org/www-project-top-10-for-large-language-model-applications/
