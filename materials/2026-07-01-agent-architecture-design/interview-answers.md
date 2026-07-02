# Agent Architecture Design — Interview Reference Answers

> 评分标准依据：Anthropic Building Effective Agents、OpenAI Agents SDK、LangGraph、Qwen-Agent、Coze、AutoGen 官方文档及 OWASP LLM Top 10 风险分类。每题标注主要依据。

---

## 1. 架构总览

题目原文：你要从 0 到 1 设计一个企业内部知识助手，它可以查文档、查工单、调用内部系统创建任务。请给出整体架构。

追问：
- 哪些部分是 workflow，哪些部分是 agent？
- 工具层怎么设计？
- 如何做权限隔离？
- 如何避免 prompt injection？

满分回答要点：
- 混合架构：RAG workflow + Agent loop + 工具侧审批
- workflow vs agent 边界清晰：确定性链路 vs 动态决策
- 工具层有 gateway 做统一管控
- 安全分层：认证 → 授权 → 数据隔离 → 输入输出过滤

示范回答（主问题）：

我会用分层混合架构。入口是 Gateway 做身份、租户、quota 和输入过滤。Router 判断请求类型——文档问答走 RAG workflow（确定性链路）、工单排查进入 Agent loop（需动态决策）、任务创建走 prepare/confirm/commit 三步（有副作用需确认）。

核心组件：RAG pipeline（向量检索 + Rerank）、Agent runtime（Planning + Tool Calling 循环）、Tool Gateway（统一管理所有内部系统调用）、Permission Service（RBAC + 数据归属校验）、Human Approval（副作用操作确认）、Trace/Eval（全链路追踪）。

示范回答（追问）：

追问 1（workflow vs agent 边界）：文档检索和知识问答是 workflow——检索→拼接→生成，路径确定。工单排查需要 agent——查到什么才能决定下一步，路径不确定。创建任务是 agent 发起但执行经 workflow——prepare 阶段可 agent 自由探索，confirm/commit 阶段是严格 workflow 不做自由决策。关键判断标准：能不能在执行前列出所有步骤？能→workflow，不能→agent。

追问 2（工具层设计）：用 Tool Gateway 统一管理——所有工具调用经过 gateway，gateway 做 schema validation、参数校验、RBAC 鉴权、限流、幂等去重、审计日志。工具定义用 JSON Schema，包含 name、description（含正向和负向引导）、parameters（类型 + 约束 + 枚举值）、risk_level（L0-L3）。内部系统通过 adapter 适配到统一 tool interface。

追问 3（权限隔离）：三层——认证层（JWT 传递用户身份）、授权层（RBAC/OPA 校验操作权限）、数据归属层（该工单/文档是否属于当前用户所在租户）。关键：Tool call 携带的是用户身份不是 Agent 身份，权限校验在 Tool Gateway 层独立完成不依赖 Agent 的判断。

追问 4（prompt injection 防护）：四层——输入层过滤（用户输入中的 system prompt 关键字标记）；工具层防护（高风险操作不在默认工具列表）；执行层校验（规则引擎独立验证 tool call 的目标和归属）；输出层（工具返回内容灌回上下文前 sanitize——工具返回是数据不是指令）。

详细解析：这道题考察宏观架构能力。面试官通过追问判断候选人是否理解"Agent 不是独立系统而是分层架构中的一个组件"。能画出分层图、说清各层职责、明确哪层做哪层不做的候选人是能独立承担架构设计的人。

常见错误：
- 只说"用 LangChain 接知识库"——太薄，没有架构分层
- 忽略权限和安全——缺乏企业级思维
- 不区分 workflow 和 agent 边界——全部用 agent 解决，成本爆炸
- 工具层没有 gateway 统一管理——每个工具各自接入

主要依据：Anthropic Building Effective Agents（workflow vs agent 边界）[来源: Anthropic]；OpenAI Agents SDK（tools/guardrails/tracing 架构）[来源: OpenAI]；OWASP LLM Top 10（excessive agency 和 prompt injection 风险）[来源: OWASP]。

---

## 2. Workflow vs Agent

题目原文：什么时候你会拒绝使用 Agent，而选择固定 workflow？请结合真实业务场景说明。

追问：
- 如果业务方坚持"要更智能"，你如何解释 trade-off？
- 什么时候可以从 workflow 演进到 agent？

满分回答要点：
- 能清晰列出选 workflow 的判断标准
- 能用成本/延迟/可靠性数据说服业务方
- 演进条件：workflow 的异常分支多到维护不下去时

示范回答（主问题）：

选 workflow 而非 agent 的场景：1）步骤可提前完整列出且不因中间结果改变——月报生成，5 个数据源拉取→计算→导出，路径固定的管道路径；2）需要合规审计——退款处理，每一步必须可追溯，不能让 agent "自由发挥"；3）错误成本极高——批量工资计算，一个 agent 的"创意"可能造成百万损失。核心判断：执行前能否完整列出所有步骤？能→workflow，不能→agent。

示范回答（追问）：

追问 1（说服业务方）：我会用数据说话。拿三个典型 case 跑对比——workflow 版本：P95 延迟 3s，成本 $0.05/次，100% 可审计；agent 版本：P95 延迟 15s，成本 $0.35/次（7x），80% 可审计（剩下 20% 要事后看 trace）。然后问："为那 20% 的'更智能'，愿意付 7x 成本和 5x 延迟、牺牲审计覆盖吗？"通常业务方看到数字就理解了。

追问 2（演进时机）：当固定 workflow 的 if-else 分支多到维护成本超过 agent 不确定性成本时——通常异常分支 > 20 个时。演进路径：先让 workflow 跑 80% 标准 case，剩下的 20% 异常 case 用 agent 处理——这是"workflow 骨架 + agent 异常处理"的混合模式，最稳。

详细解析：这道题考察候选人的工程判断力——不是"会用 agent"而是"知道什么时候不该用"。面试官最想听到的是具体的量化判断标准和说服业务方的方法论，而不是技术名词。

常见错误：
- "agent 永远比 workflow 好"——不理解复杂度和成本
- 只说概念不谈数据——无说服力
- 不提供演进路径——要么 agent 要么 workflow 的二选一思维

主要依据：Anthropic Building Effective Agents（workflow/agent 区分和复杂度取舍）[来源: Anthropic]；生产 Agent vs Workflow 成本对比实践 [来源: FutureAGI]。

---

## 3. 工具调用与副作用

题目原文：设计一个 Agent 帮销售自动更新 CRM、生成邮件、安排会议。哪些操作可以自动执行，哪些需要用户确认？

追问：
- 如何保证工具幂等？
- 如何处理工具超时和部分成功？
- 如何审计一次工具调用链？

满分回答要点：
- 按副作用分层：只读→自动，新增→软确认，修改→硬确认，删除/发送→人工审核
- 幂等：每条变更带 idempotency_key
- 审计：每步 tool call + result + timestamp 记录

示范回答（主问题）：

四级分层——自动执行：查询 CRM 数据、查日历空闲时段、生成邮件草稿（只读/预览类）。用户软确认（UI 点击确认）：更新 CRM 字段、保存草稿。用户硬确认（验证码）：创建新联系人、安排会议（影响他人时间）。人工审核：批量群发邮件、删除 CRM 记录（不可逆/广播类）。

示范回答（追问）：

追问 1（幂等保证）：每条变更操作带 idempotency_key（UUID），后端在执行前先查"这个 key 是否已执行过"——已执行返回原结果不重复操作。重试场景下 Agent 可能多次调用同一 tool，幂等 key 防止 CRM 里重复创建联系人。生成规则：idempotency_key = hash(session_id + step_id + tool_name + arguments)。

追问 2（超时和部分成功）：超时——读操作重试（指数退避 2 次），写操作切异步轮询（返回 job_id）。部分成功——CRM 更新了但邮件没发出去，不要静默。Agent 应收到"CRM 成功（字段 X/Y/Z 已更新），邮件失败（SMTP timeout），建议补充行动"的结构化状态。Agent 据此决定"通知用户 CRM 已更新、邮件需手动重发"还是"全部回滚"。

追问 3（审计链）：每步 tool call 记录 trace_id、step_id、tool_name、arguments、result、timestamp、user_id、session_id。结构化存储（非纯文本日志）支持按 trace_id 回放整条调用链。高风险操作（delete/send）额外记录审批人、审批时间、审批决策。

详细解析：这道题考察对"Agent 是危险的工具使用者"的理解。面试官想知道候选人是否理解不同副作用的风险差异，以及是否在工具层而非 Agent 层做防护。

常见错误：
- 所有操作都需要确认——繁琐不实用
- 幂等只靠"同名工具去重"——参数细微不同就失效
- 审计只记"调了什么工具"——不记录具体的参数和结果

主要依据：OWASP LLM Top 10（excessive agency 和副作用的权限控制）[来源: OWASP]；OpenAI Agents SDK（tools 的 guardrails 和 approval 机制）[来源: OpenAI]；Anthropic Tool Use（权限和副作用的最佳实践）[来源: Anthropic]。

---

## 4. 状态与记忆

题目原文：一个长任务 Agent 需要连续运行 30 分钟，中间可能等待用户确认，也可能工具失败重试。你如何设计状态管理？

追问：
- 状态存在数据库、Redis 还是向量库？
- 长期记忆和 execution state 有什么区别？
- 如何恢复中断任务？

满分回答要点：
- execution state 和 memory 是两种不同的存储需求
- 状态恢复需要 checkpoint + 幂等重放
- 长期记忆用向量库，执行状态用 DB/Redis

示范回答（主问题）：

两层状态管理。Execution State（运行时状态）——当前任务到哪一步了、哪些工具已调用/成功/失败、等待用户确认哪个操作。存在持久化存储（Postgres）保证进程重启不丢。Memory（长期记忆）——用户的偏好和历史的决策结果，存在向量库供后续任务检索。中间用 Redis 做热状态缓存。

示范回答（追问）：

追问 1（存储选型）：Execution state 存 Postgres——需要事务性、精确查询、不丢数据。热状态（当前步、pending approvals）走 Redis——低延迟读写、带 TTL 自动过期。长期记忆走向量库——语义检索"类似的工单上次怎么处理的"。三者各司其职不混用。

追问 2（execution state vs 长期记忆）：Execution state 是 "当前任务的具体运行状态"——第 3 步查了工单返回了 ID#4521、第 4 步等待用户确认。长期记忆是 "跨任务的持久知识"——用户喜欢简短回复、上次退款处理用了加急通道。前者像函数调用栈，后者像数据库。

追问 3（中断恢复）：每个步骤完成后写 checkpoint（step_id + 输入 + 输出 + 状态）。恢复时从最后一个成功 checkpoint 后继续，配合幂等重放避免重复执行已完成步骤。用户确认类操作在恢复时重新询问（上次的确认可能已过期）。

详细解析：状态管理是长任务 Agent 最容易出问题的地方——进程重启丢失状态是最常见的线上事故之一。面试官通过追问判断候选人是否真正做过长任务系统。

常见错误：
- 状态全存 Redis——重启就丢
- 不区分 execution state 和 memory——混在一起导致概念混乱
- 恢复中断时从头重做——浪费且可能重复副作用操作

主要依据：LangGraph Durable Execution（checkpoint 和状态恢复机制）[来源: LangGraph Docs]；MemGPT Paper（长期记忆和上下文管理）[来源: arXiv]。

---

## 5. 框架选型

题目原文：现在有 LangGraph、OpenAI Agents SDK、Qwen-Agent、Coze、AutoGen。你会如何为一个面向国内企业客户的 Agent 产品选型？

追问：
- 如果必须私有化部署呢？
- 如果目标是快速验证业务呢？
- 如果目标是复杂长任务可靠执行呢？

满分回答要点：
- 选型维度：部署方式、生态绑定、长任务可靠性、国内合规
- 不同目标对应不同选择，不是"哪个最好"
- 能给出具体决策树

示范回答（主问题）：

面向国内企业客户，关键约束是私有化部署 + 信创合规 + 国内模型生态。这个约束下大幅收窄选择——优先 Qwen-Agent（原生支持 Qwen 系列、阿里云部署、国内合规）或 LangGraph（开源可控、自部署、模型无关）。OpenAI Agents SDK 和 Coze 在私有化和国内合规上有劣势。AutoGen 适合多 Agent 研究场景但生产稳定性待验证。

示范回答（追问）：

追问 1（私有化部署）：排除 Coze（纯 SaaS）和 OpenAI Agents SDK（GPT 绑定）。选 LangGraph——开源、模型无关（可接 Qwen/GLM/DeepSeek）、自部署可控、Python 生态完整。备选 Qwen-Agent 如果全栈阿里云。

追问 2（快速验证）：选 Coze 或 Dify——最快 1 天内搭出可用 demo、拖拽式 workflow 编辑器、内置 RAG 和 tool 接入、无需写代码。验证了 PMF 后再考虑工程化迁移到 LangGraph。

追问 3（复杂长任务）：选 LangGraph——graph-based 状态管理、checkpoint/persistence、human-in-the-loop、streaming。AutoGen 也支持多 Agent 但复杂长任务的 checkpoint 和恢复机制不如 LangGraph 成熟。Qwen-Agent 在长任务稳定性上还在快速迭代。

详细解析：框架选型是面试高频题——面试官想看候选人是否有"根据不同约束选择不同工具"的能力，而非"我就用这个"。能给出决策树而非单一答案的候选人 fit 架构角色。

常见错误：
- 只推荐自己会用的框架——不考虑场景约束
- 忽略国内私有化/合规要求——照搬海外方案
- 不考虑快速验证和工程化的区别——一刀切

主要依据：LangGraph 官方文档（durable execution 和 checkpoint）[来源: LangGraph Docs]；Qwen-Agent GitHub（国内模型生态定位）[来源: Qwen-Agent GitHub]；OpenAI Agents SDK（GPT 生态绑定和托管服务）[来源: OpenAI]；Coze 官方文档（低码拖拽平台定位）[来源: Coze Docs]；AutoGen 官方文档（多 Agent 框架定位）[来源: Microsoft AutoGen]。

---

## 6. 评测与线上问题定位

题目原文：你的 Agent 上线后用户投诉"经常答非所问，有时乱调用工具"。你会怎么建立评测和观测体系？

追问：
- 需要哪些离线测试集？
- trace 里必须记录什么？
- 如何做回归测试？

满分回答要点：
- 三维评测：离线测试集 + 线上 trace + 用户反馈闭环
- trace 记录：每步 tool call + 参数 + 结果 + 决策 + 延迟
- 回归：golden dataset + 每次发版跑 diff

示范回答（主问题）：

三维评测体系。离线评测：构建 Golden Dataset（100-500 条真实用户请求 + 标注的正确 tool call 路径 + 预期最终回复）。每次改 prompt 或换模型后跑 eval，对比 tool 选择准确率、参数 F1、端到端成功率的变化。线上追踪：OpenTelemetry 记录每步 tool_call + arguments + result + thought + latency，标记异常（tool 选择变更、延迟突增、用户负面反馈）。用户反馈闭环：用户点"踩"的 conversation 自动入池，每周 review top bad cases 加入 golden dataset。

示范回答（追问）：

追问 1（离线测试集）：四类——1）Happy path：标准请求，正确答案明确；2）边界 case：模糊输入、多工具可选；3）对抗样本：prompt injection 尝试、异常参数；4）回归集：历史上线 bug 的 conversation。来源于生产日志 + 人工标注 + 对抗构造。覆盖工具选择、参数填充、不必要调用、结果整合四步。

追问 2（trace 记录）：必须记录——trace_id、step_id、timestamp、user_input、model_decision（为什么选这个工具）、tool_name、arguments、tool_result、latency_ms、success/error_code、retry_count、final_response。结构化存储（JSON 字段）支持按 tool_name、error_code、latency 聚合查询。

追问 3（回归测试）：维护 Golden Dataset 50-200 条。每次改 prompt/schema/模型版本时跑完整 eval——对于每条 golden case：新版本输出的 tool call 是否和标准答案一致、最终回复的核心信息是否一致。CI 集成回归测试，关键指标（工具选择准确率、端到端成功率）下降 > 3% 阻塞发版。

详细解析：这是生产 Agent 最被低估的环节。面试官想看候选人是否有"评测量化"的意识——不是"感觉改好了"，而是"数据说改好了"。trace 设计是区分"做 demo"和"做产品"的关键。

常见错误：
- 没有离线测试集——全靠"上线看看"
- trace 只记了调用不计参数和结果——无法复现
- 回归测试做成"发版前跑一下"——应该 CI 集成有阈值

主要依据：OpenAI Agents SDK（tracing 和 guardrails 体系）[来源: OpenAI]；LangGraph Production Runtime（checkpoint 和 trace 机制）[来源: LangGraph Docs]；BFCL v3 评测方法论（四步评估框架）[来源: FutureAGI]。

---

## 7. 成本与延迟

题目原文：一个 Agent 平均要调用 5 次模型、3 次检索、2 次工具，P95 延迟超过 20 秒。你如何优化？

追问：
- 哪些步骤可以并行？
- 如何做模型路由？
- 如何设置 budget 和 early stop？

满分回答要点：
- 三步优化：并行化 → 模型路由 → budget 控制
- 独立调用并行化是第一刀
- 模型路由省钱、budget 控制兜底

示范回答（主问题）：

三步优化。第一步并行化——识别无数据依赖的调用：3 次检索如果查询不同源可并行、2 次工具互不依赖也并行。原来是 5+3+2=10 步串行，优化后 5+max(3,2)+1=8 步，减少 ~20% 延迟。第二步模型路由——简单步骤（检索结果总结、参数填充）用 Haiku/4o-mini，只有复杂决策（工具选择、异常处理）用强模型。可降低 40-60% 成本。第三步 budget 控制——设 max_steps=10、total_timeout=15s、token_budget=8000，超了强制输出当前最佳答案。

示范回答（追问）：

追问 1（并行化）：检索层——多个数据源（文档库 + 工单库 + 知识图谱）可同时检索。工具层——查询 CRM + 查询日历（互不依赖）可并行。LLM 推理层——如果架构允许，简单推理步骤可批处理。关键是用依赖图标记而不是人工判断——工具 A 需 B 的输出则标记依赖，否则默认并行。

追问 2（模型路由）：三级路由——查询类（"订单状态是什么"）直接用 cheap 模型调工具返回结果；分析类（"为什么这个月退货率高了"）用 mid 模型做多步推理；异常类（工具连续失败、结果矛盾）升级到 best 模型。路由决策基于请求特征（复杂度分类器）+ 运行时信号（连续失败升级）。

追问 3（budget 和 early stop）：max_steps=10（超过强制输出当前已知信息）。total_timeout=15s（从用户请求开始计时）。token_budget=8000（超过开始压缩旧上下文）。early stop 信号：连续 3 步无实质进展（没获取新信息/没缩小问题/没改变方向）、工具返回了可直接回答用户的完整信息。

详细解析：成本和延迟是决定 Agent 能否上生产的关键指标。面试官通过追问判断候选人是否能从系统层面优化而不仅仅是"换更快的模型"。

常见错误：
- 只知道 prompt cache 和模型切换——不懂并行优化
- 不设 budget 和 early stop——Agent 失控烧钱
- 模型路由没有降级策略——强模型挂了就不行

主要依据：Anthropic Building Effective Agents（成本控制和简洁原则）[来源: Anthropic]；OpenAI Agents SDK（model routing 和 tracing）[来源: OpenAI]；生产 Agent 成本优化实践 [来源: FutureAGI]。

---

## 8. Multi-Agent 取舍

题目原文：业务方提出"我们做一个多个专家 Agent 协作的系统，一个负责检索，一个负责规划，一个负责执行，一个负责评审"。你如何评价这个方案？

追问：
- multi-agent 的收益是什么？
- 风险是什么？
- 什么情况下你会真的采用？

满分回答要点：
- 加 Agent = 加复杂度 + 加 token + 加失败概率
- 收益：专业分工、交叉验证、并行处理
- 风险：通信 overhead、幻觉放大、调试困难
- 采用条件：任务天然可并行、需要多视角验证、单 Agent 已优化到极限

示范回答（主问题）：

我会先问业务方："用一个强 Agent 跑过吗？效果哪里不满意？"因为大多数场景下一个强 Agent 反而更稳。四个 Agent 的架构看似分工清晰，但每个 Agent 之间的通信都是 token 开销、每个 Agent 都可能引入自己的幻觉、调试时四个 Agent 的行为交叉极难定位。我的原则是——先单 Agent，优化到瓶颈再考虑拆。如果是知识助手场景，一个 Agent + 好的 Tool Schema 通常足够。

示范回答（追问）：

追问 1（收益）：1）专业分工——检索专家用专门的检索策略，评审专家有严格的质量标准，比一个 Agent 面面俱到好。2）交叉验证——评审 Agent 抓到执行 Agent 的错误时可以做 self-correction。3）并行处理——检索和规划可以并行启动。但这些收益都有前提——任务复杂度够高。

追问 2（风险）：1）幻觉放大——检索 Agent 编造了一个文档引用，规划 Agent 基于它做了计划，执行 Agent 执行了错误计划，评审 Agent 如果是同一模型的会确认"没问题"。四个 Agent 串联放大了单点幻觉。2）通信 overhead——Agent 之间要传 context，4 个 Agent 的 context 量是 1 个 Agent 的 3-4 倍。3）调试地狱——用户反馈"答案不对"，需要排查四个 Agent 的交互链路，比单 Agent trace 复杂一个量级。

追问 3（采用条件）：三个条件同时满足——1）单 Agent 已经优化到极限（prompt、schema、tools 都精细调过了）仍然不够；2）任务天然需要多视角（比如法律 + 技术 + 合规三方交叉审查）；3）有足够的工程资源去做多 Agent 的 trace、debug、eval infrastructure。三者缺一不可。强制条件：至少有一个 Agent 是"评审/验证"角色作为质量 gate。

详细解析：这道题考察对复杂度的敬畏。面试官想看候选人是否理解"加 Agent 不是加功能是加复杂度"。能说出 Multi-Agent 具体风险（幻觉放大、调试困难）的候选人是真做过的。

常见错误：
- "四个 Agent 很好，专业分工"——不理解复杂度代价
- 不评估单 Agent 是否够——上来就拆
- 不提调试困难——缺乏生产运维经验

主要依据：Anthropic Building Effective Agents（orchestrator-workers 模式和多 Agent 取舍）[来源: Anthropic]；AutoGen Multi-Agent 论文（协作模式的风险和收益）[来源: Microsoft AutoGen]；FutureAGI Agent Architecture Patterns 2026（多 Agent 生产实践）[来源: FutureAGI]。

---

## 来源

- [Anthropic: Building Effective Agents](https://www.anthropic.com/engineering/building-effective-agents)
- [OpenAI Agents SDK Documentation](https://openai.github.io/openai-agents-python/)
- [LangGraph Documentation](https://docs.langchain.com/oss/python/langgraph/overview)
- [Microsoft AutoGen Documentation](https://microsoft.github.io/autogen/)
- [Qwen-Agent GitHub](https://github.com/QwenLM/Qwen-Agent)
- [Coze Documentation](https://www.coze.cn/open/docs/guides/welcome)
- [OWASP Top 10 for LLM Applications](https://owasp.org/www-project-top-10-for-large-language-model-applications/)
- [FutureAGI: Agent Architecture Patterns 2026](https://futureagi.com/blog/agent-architecture-patterns-2026/)
