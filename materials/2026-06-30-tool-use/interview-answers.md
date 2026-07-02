# Tool Use / Function Calling — Interview Reference Answers

> 评分标准依据：FutureAGI 2026 Four-Step Evaluation、OWASP LLM Top 10、Anthropic Tool Use Documentation、字节/阿里 Agent 面试评分要点。每题标注主要依据。

---

## 1. LLM Tool Use 完整生命周期

题目原文：请解释 LLM Tool Use 的完整生命周期——从用户发消息到模型回复，中间发生了什么？不要只讲 happy path，把可能的异常分支也讲进去。

追问：
- 如果执行超时了怎么办？5 秒和 30 秒超时策略有何不同？
- 模型的 token 生成怎么知道该在哪里停？训练的还是 prompt 的？
- 同时决定调用 3 个工具，引擎串行还是并行？怎么判断？

满分回答要点：
- 能画出完整状态流转图（意图识别 → 决策 → 执行 → 注入 → 输出）
- 超时分层：短超时重试、长超时切异步
- 训练决定 vs prompt 决定
- 并行判断标准：数据依赖而非读写权限

示范回答（主问题）：

完整生命周期分五步。第一步意图识别——模型收到 user message + system prompt（含 tool definitions），在普通文本和 tool call 两种输出模式间切换，切换能力来自微调训练。第二步结构化输出——模型输出 tool call（函数名 + JSON 参数），以 OpenAI 为例输出 `tool_calls` 数组含 `id`、`function.name`、`function.arguments`，框架解析器识别特殊格式并拦截。第三步工具执行——框架调用对应的 Python/JS 函数，纯工程层。第四步结果注入——工具返回值以 `role: "tool"` 追加到对话历史。第五步最终输出——模型综合结果生成自然语言回复。

异常分支：超时 → 5 秒快速重试，30 秒切异步轮询；解析失败 → error message 喂回模型让其 self-correct；工具不存在 → 明确错误不重试；权限不足 → 告知模型转化为用户友好的"抱歉没有权限"。

示范回答（追问）：

追问 1（超时策略）：5 秒超时说明是瞬时故障（网络抖动），做指数退避重试（1s→2s→4s），通常 2-3 次能恢复。30 秒超时说明工具本身处理慢——不要再重试，返回 job_id 切成异步轮询模式，让模型每隔 N 秒查一次任务状态。超时阈值应在框架/工具定义层设定，不应让模型传入。

追问 2（在哪停）：是训练决定的。模型在 SFT/RL 阶段学习了大量 tool call 格式的对话数据，学会了在需要外部信息时输出特殊 token（如 `<tool_call>` 或 `tool_calls` 字段），这个 token 触发框架拦截。Prompt 可以提供工具 schema 但无法从零教会这个能力——预训练和 fine-tuning 阶段的工具语料才是关键。

追问 3（并行判断）：同一轮推理中模型输出的多个 tool_call，默认之间没有数据依赖 → 并行执行。判断标准是数据依赖不是读写权限——两个互不依赖的写操作也可以并行。有依赖的情况（先查订单再查物流）天然由推理轮次分隔——第一轮查订单，结果回注后再决定查物流，跨轮即串行。

详细解析：这道题考察的是对 Tool Use 底层机制的理解深度。面试官区分"会用 API"和"理解原理"的关键点在于：1）能否说清楚每步谁负责（模型 vs 框架）；2）异常分支是否分层处理而非一刀切；3）并行判断标准是否正确（最常见错误就是按读写权限判断）。字节面试极其喜欢追问超时和并行这两点。

常见错误：
- 说工具是"模型内部执行的"——混淆决策层和执行层
- 不讲异常分支——happy path only
- 并行判断按"读写权限"而非"数据依赖"
- 说 tool call 是 prompt 写出来的——低估训练的作用

主要依据：OpenAI Function Calling Guide（模型输出格式和生命周期）[来源: OpenAI Docs]；Anthropic Tool Use Documentation（工具调用的角色分离）[来源: Anthropic Docs]；字节 Agent 面试真题 2026（超时和并行追问）[来源: 华为云博客]。

---

## 2. Schema 设计与语义理解

题目原文：设计 `search_users` 和 `get_user` 两个工具，description 怎么写才能让模型在"模糊搜索"和"精确查询"之间做出正确选择？

追问：
- "帮我找一下张三"——模型该用哪个？选错了改哪里？
- description 写"用于搜索用户"就够了？具体需要什么？
- search_users 返回 50 个结果怎么处理？
- 如何衡量 tool schema 设计得好不好？量化指标？

满分回答要点：
- 两个工具的 description 必须互斥且有决策边界
- 负向引导（"什么时候不找我"）和正向引导同等重要
- 50 结果场景要给出简洁模式 + 阈值分段 + 主动缩小范围
- 能量化（工具选择准确率、参数 F1、不必要调用率）

示范回答（主问题）：

核心原则：description 是模型决定"该不该调"和"选哪个"的唯一信号。两个工具的 description 必须互斥：

search_users：模糊搜索。当用户条件不精确时（"找一下张三"、"姓李的"）使用。返回匹配列表（id + 姓名 + 部门），最多 20 条。找到后如需详情再调 get_user。不要对已知 ID 的查询使用此工具。

get_user：精确查询。只在已知 user_id 时使用。如果没有确切 ID，先用 search_users。不要用此工具做模糊搜索。

关键是双向引导——一个说"拿到 ID 后用那个"，另一个说"没有 ID 就先去那边"。

示范回答（追问）：

追问 1（"找一下张三"）：用 search_users(query="张三")。因为用户没给确切 ID。如果模型选错了 get_user，说明 description 缺少负向引导——get_user 的 description 没写"不要用于模糊查询"。在 description 中加排他性边界即可修复。

追问 2（description 的充分性）："用于搜索用户"远远不够。完整 description 需包含：1）工具的功能（做什么）；2）什么时候用（正向引导）；3）什么时候不用（负向引导，这个最容易被忽略但最有效）；4）参数的业务含义而非技术名；5）返回格式和下一步建议。

追问 3（50 结果处理）：不要直接塞 50 个 JSON 给模型。三层策略：1）简洁模式——只返回 name + ID + 关键描述字段；2）阈值分段——超过 N 个结果自动切换简洁模式；3）主动缩小——模型回复"找到 50 个匹配，请提供部门或其他信息来缩小范围"。

追问 4（量化指标）：四个指标——1）工具选择准确率（选对工具的占比，需人工标注 ground truth）；2）参数填充 F1（必填参数的完整率和准确率）；3）不必要调用率（该直接回答却调了工具的占比）；4）端到端成功率（用户满意的比例，最终目标）。

详细解析：这道题考察的是"Schema 即 Prompt Engineering"的认知深度。面试官通过追问判断候选人是否真做过工具设计——真做过的人一定会提负向引导和决策边界，没做过的人只会说"写清楚"。字节尤其喜欢追问"50 个结果怎么处理"，考察大规模返回值的工程意识。

常见错误：
- description 太短太笼统——"用于搜索用户"确实不够
- 没说清楚工具间的调用顺序依赖（双向引导）
- description 没有负向引导（"什么时候不找我"）
- 没有考虑返回结果过多的情况——缺乏工程经验

主要依据：FutureAGI Evaluating LLM Tool Use 2026（四步评估框架）[来源: FutureAGI]；Skywork AI Ultimate Guide 2026（description 最佳实践）[来源: Skywork AI]；字节 Agent 面试题"给 Agent 设计工具"（Schema 设计原则）[来源: 华为云博客]。

---

## 3. 可靠性工程

题目原文：Tool Use 在生产环境中的失败率通常在 3-10%。请设计一套容错体系，目标是让用户看不到任何工具调用失败。

追问：
- 重试的退避策略是什么？什么时候不该重试？
- 重试 3 次都失败后 fallback 怎么做？
- 工具返回数据格式和 Schema 不一致怎么处理？
- 如何监控工具调用质量？需要哪些 metrics？

满分回答要点：
- 四层容错体系：校验 → 重试 → 熔断 → 降级
- 显式区分可重试和不可重试错误
- fallback 有缓存降级路径不是只报错
- 监控面要覆盖首调成功率、重试率、错误分布

示范回答（主问题）：

四层容错体系。L1 客户端校验：发 tool call 前验证参数格式，Schema 级别类型检查，不合规直接拦截并返回错误信息给模型修正。L2 重试策略：可重试（网络超时/503/429）→ 指数退避 1s→2s→4s 最多 3 次；不可重试（参数错误 400/权限 403/资源不存在 404）→ 直接返回 error 给模型让它重新规划。L3 熔断降级：连续失败 N 次后熔断该工具，走替代工具或通用兜底话术。L4 Schema 漂移防御：loose parse 匹配，允许未知字段，关键字段缺失时走 fallback。

示范回答（追问）：

追问 1（退避策略）：指数退避 1s→2s→4s，最多 3 次。关键是不该重试的情况——参数错误（400）重试无意义、权限不足（403）重试不会改变、资源不存在（404）同样。业务错误（"库存不足"）不是系统故障，直接告知用户不重试。

追问 2（fallback）：不是简单报错。三层降级——1）有替代工具就走替代（查不到物流查订单状态）；2）无替代走优雅降级话术（"XX 服务暂不可用，已记录您的问题"）；3）查询类可返回缓存或上次成功结果。绝对不展示 stack trace 或 raw JSON error。

追问 3（Schema 漂移）：三层防御——1）解析时做 loose parse，用 schema 子集匹配允许未知字段；2）对关键字段类型检查，非关键字段缺失不阻断；3）完全无法解析时标为"工具异常"走 L3 降级。监控字段变更频率预警。

追问 4（监控指标）：首调成功率（first-attempt success，衡量模型生成质量）、调用成功率（最终成功/总调用，分工具看）、重试率、错误分布（按类型占比）、端到端延迟 P50/P95/P99（按工具分）、用户满意度（点赞/踩率）。"调用成功率"的定义需要排除业务错误——只算技术层面成功调用。

详细解析：这道题区分"写代码的"和"做生产的"。面试官通过追问判断候选人是否经历过线上故障——真经历过的人一定会区分可重试/不可重试错误、会提熔断、会算成本。阿里尤其喜欢追问"重试 3 次都失败后怎么办"——考察降级 gracefulness。

常见错误：
- 所有错误都重试——业务错误重试无意义
- 没有降级路径——fallback 是"报错"不够
- 不讲监控——缺乏生产意识
- 定义"调用成功率"不排除业务错误——指标设计不准

主要依据：Cruxial 可靠性层设计 2026（auto-repair 分类体系）[来源: Cruxial GitHub]；Kalvium Labs 生产 Agent 数据 2026（14% vs 2.1% 失败率）[来源: Kalvium Labs]；Self-Reflective APIs 论文 June 2026（结构化错误恢复）[来源: arXiv 2606.05037]；字节 Agent 面试题 2026（异常处理追问）[来源: 华为云博客]。

---

## 4. Multi-Turn & 上下文管理

题目原文：用户和 Agent 进行了 15 轮对话，其中调用了 6 次工具。到了第 15 轮，上下文窗口快满了。你怎么设计上下文管理？

追问：
- 工具返回的原始 JSON 和模型重新组织的回复——两者都要保留吗？
- 怎么判断某次工具调用结果后续还需要？
- 设计一个工具结果"摘要器"怎么做？风险？
- 这和 KV Cache 管理有什么关系？

满分回答要点：
- tool_call 保留（推理链需要），tool_response 可压缩
- 判断标准：后续引用 → 依赖图，而非时间远近
- 摘要器要结构化提取关键字段，风险是丢关键信息
- 上下文瘦身 → KV Cache 等比缩小

示范回答（主问题）：

三层策略。1）滑动窗口 + 摘要——最近 N 轮保留完整，更早的 tool_response 用摘要器压缩为关键字段。2）依赖图判断——构建依赖图，后续轮次引用过的 tool call 结果保留，孤立节点优先压缩，比单纯看时间远近更准确。3）分层存储——当前任务上下文（热）→ Session 内关键信息（温）→ 历史摘要（冷）。

示范回答（追问）：

追问 1（哪些保留）：tool_call 保留因为模型需要维护"我调了什么"的推理链。tool_response 可以压缩——模型关心的是关键字段不是完整 JSON 结构。模型对 tool_response 的自然语言总结保留——这是模型后续推理的核心上下文。

追问 2（依赖图判断）：比时间远近更准的方法是构建依赖图——每次 tool call 的结果被后续哪几轮引用过。有子孙引用的保留原始结果，孤立节点直接压缩为一行摘要。Anthropic 和 OpenAI 的长对话 agent 已在用类似技术。

追问 3（摘要器设计）：成功返回 → 只保留关键字段如 `{"order_id": 123, "status": "shipped"}`，丢弃元数据。失败返回 → 保留错误类型和 message，丢弃 stack trace。风险：摘要可能丢弃一个看似无关的字段但下轮变得关键。缓解：对摘要后上下文做 key-value 索引，可回溯。

追问 4（KV Cache 关系）：KV Cache 大小正比于上下文长度。减少上下文中工具结果的 token 数 → KV Cache 等比缩小。上下文管理和 KV Cache 管理是同一枚硬币的两面。

详细解析：这题区分"用过 API"和"跑过生产"——真跑过生产的人遇到过长对话 OOM 或 tokens 超限，一定会有上下文管理策略。面试官通过追问"依赖图"判断候选人是否做过系统设计层面的思考。

常见错误：
- 说"全保留"——不考虑成本
- 说"全丢掉"——不考虑后续引用
- 不分 tool_call 和 tool_response——两者保留策略不同
- 不知道 KV Cache 和上下文管理的关系

主要依据：Anthropic 长对话 Agent 实践（依赖图管理上下文）[来源: Anthropic Docs]；vLLM PagedAttention（KV Cache 与上下文长度关系）[来源: vLLM Docs]；生产 Agent 上下文管理实践 2026 [来源: Kalvium Labs]。

---

## 5. 安全与权限

题目原文：多租户 SaaS 系统中，Agent 可调用 `delete_user`、`query_order` 等敏感工具。如何保证 Agent 不会越权操作？

追问：
- Tool call 携带用户身份还是 Agent 身份？权限校验在哪层？
- delete_user 没设 user_id 参数（隐式当前用户）的安全风险？
- prompt injection 诱导调用不该调的工具怎么防护？
- 设计 Human-in-the-Loop 机制，哪些操作需确认？

满分回答要点：
- Tool call 携带用户身份，Agent 只是代理
- 三层校验：认证 → 授权 → 业务归属
- 显式 user_id > 隐式，避免 prompt injection 删除自身
- HITL 分层：自动 → 软确认 → 硬确认 → 人工审核

示范回答（主问题）：

Tool call 携带的是用户身份不是 Agent 身份。Agent 只是代理——用户说"删我的订单"，Agent 调用 cancel_order(order_id)，权限校验应检查当前用户是否有权取消这个订单。三层校验：认证层（AuthN——验证是谁）、授权层（AuthZ——验证有权执行该操作）、业务层（验证数据归属——这个订单是当前用户的吗）。

示范回答（追问）：

追问 1（身份和校验层）：Tool call 携带用户身份（JWT/session token）。Agent 是代理不是主体。校验在工具执行前由独立权限服务完成——不在 Agent 层做，Agent 层只负责生成调用意图。

追问 2（隐式 user_id 风险）：如果 delete_user 没提供 user_id（隐式用当前用户），最大风险是 prompt injection——攻击者诱导 Agent 调用 delete_user，当前用户就被删了。安全设计：所有变更类操作强制传 user_id，后端校验 user_id == current_user_id OR is_admin，不允许隐式代理。

追问 3（prompt injection 防护）：三层防护——1）输入过滤：用户输入中的 system prompt 关键字标记权重；2）工具白名单：高风险操作（delete、transfer）不在默认工具列表，需额外确认；3）独立校验层：即使模型输出了 tool call，执行前由规则引擎独立校验"模型说删谁、用户是本人吗、目标属于用户吗"。工具返回结果灌回上下文前 sanitize——工具返回的也是数据不是指令。

追问 4（HITL 设计）：四级分层——L0 自动通过（查询本人数据）；L1 软确认（修改本人数据，UI 弹窗显示即将执行的操作）；L2 硬确认（删除/支付/权限变更，需验证码或二次认证）；L3 人工审核（金额 > 阈值/批量操作/异常模式，进入审核队列）。

详细解析：安全是 Agent 面试的必考题，字节和阿里都考。面试官通过追问判断候选人是否真正理解"Agent 不是用户本人"——真理解的人会提三层校验和显式参数，不理解的人会说"加个权限检查就行了"。prompt injection 防护是 2026 年面试新热点。

常见错误：
- 把认证和授权混为一谈
- 忽略 prompt injection 攻击面
- 所有操作都需确认——影响 UX，应分层
- Tool call 说是 Agent 身份——应该是用户身份

主要依据：OWASP LLM Top 10（excessive agency 风险分类）[来源: OWASP]；Anthropic Tool Use Security（工具权限和 prompt injection 防护）[来源: Anthropic Docs]；OpenAI Agents SDK（guardrails 和 HITL 设计）[来源: OpenAI Docs]；字节/阿里 Agent 安全面试题 2026 [来源: 华为云博客]。

---

## 6. Agent 自主性与编排

题目原文：当前 LLM 的工具编排能力最大局限是什么？举个翻车的例子。

追问：
- ReAct 和纯 Tool Use 区别？ReAct 更适合什么？
- Agent 循环调用怎么打断？
- 怎么在 3 轮内判断 Agent 是在推理还是瞎转？
- 工具编排靠模型能力还是工程约束？

满分回答要点：
- 核心局限：局部最优无全局规划，不会回溯
- 翻车例子：多订单场景只看第一个就下结论
- 循环打断：重复检测 + 语义循环 + max turns
- 3 轮判断：是否有新信息/缩小问题范围/改变方向
- 长期：模型能力 + 工程约束两手抓

示范回答（主问题）：

最大局限是"每步做局部最优决策，没有全局规划"。模型不能像 A* 搜索那样从目标反推需要什么工具，只能一步步往前试。表现：短视（选当前最好的走几步后死胡同）、不回溯（走错不会回分支点）、循环（换参数反复调期望结果改变）。翻车例子：用户说"上周买了三样东西有一个还没到"，Agent 查到 3 个订单→随机挑一个查物流→发现已签收→回复"您的订单都已签收"。应该是逐一查物流再报告具体哪个没到。

示范回答（追问）：

追问 1（ReAct vs 纯 Tool Use）：纯 Tool Use 模型直接输出 tool call 无中间推理。ReAct 模型先输出 Thought（"我需要先..."）再输出 Action，Thought 帮助厘清思路减少短视。ReAct 更适合多步推理和分支判断的复杂场景，纯 Tool Use 适合 1-3 步简单任务。

追问 2（打断循环）：三层——1）重复检测：连续 2 次同一工具 + 同一参数 → 阻止第三次；2）语义循环检测：即使参数不同，连续 N 次同类工具"没找到"视为循环；3）Max turns 硬限制（10-15 步），超了终止。

追问 3（3 轮判断法）：判断标准——3 轮后是否获取到新信息、是否缩小了问题范围、是否改变了行动方向。三者全否则大概率在瞎转。实质进展 = 每轮 tool_response 的信息量递增或问题空间缩小。不是看步数看信息增益。

追问 4（模型 vs 工程）：两手抓不站队。模型侧：更强 planning（o1-style 思考链 + 工具调用）和全局规划。工程侧：状态机 + Guardrails——关键流程不依赖模型自由决策而限定在预定义流程图中。模型越强工程约束可以越松，但永远需要兜底。

详细解析：这题考察深度思考能力——面试官不要标准答案，要看候选人是否真正思考过编排的本质局限。能举出翻车例子是区分点——真做过 agent 的人一定遇到过编排翻车。追问趋势考察 ReAct 认知和工程防护思路。

常见错误：
- 高估模型能力——"模型能自己想明白"
- 不提实际翻车例子——缺乏生产经验
- 在"模型 vs 工程"上站一边——应该是互补
- 不知道如何区分"推理"和"瞎转"——缺少判断框架

主要依据：Yao et al. ReAct 论文 2022 [来源: arXiv:2210.03629]；字节面试题"Agent 调了三个工具就死循环了"（循环打断设计）[来源: 华为云博客]；FutureAGI Agent Architecture Patterns 2026（编排局限和混合模式）[来源: FutureAGI]。

---

## 7. 模型选型与成本

题目原文：模型 A 准确率 95%/800ms/$0.03，模型 B 准确率 88%/200ms/$0.003。给小流量客服和大流量搜索分别怎么选？为什么？

追问：
- 大流量选便宜模型但 12% 失败怎么算总成本？
- 让模型 A 只处理模型 B 搞不定的 case 怎么工程化？
- 微调小模型做 Tool Use 需要什么数据？从哪来？
- BFCL 评测和真实场景的 gap 是什么？

满分回答要点：
- 客服选 A（容错空间小，失败代价远超模型成本差）
- 搜索选 B（薄利高量，用户容忍重试，成本差 10x 难靠准确率弥补）
- routing 架构：置信度阈值 + speculative routing
- BFCL 局限：单轮为主、无延迟约束、无对抗样本

示范回答（主问题）：

小流量客服选模型 A。客服场景单次失败可能转化人工或流失，一个客服坐席年薪 30 万，模型日处理 1000 次对话成本不到 $10，成本差异在整体业务中可忽略。大流量搜索选模型 B。搜索每请求利润极薄，用户对重试容忍高。100 万次搜索：模型 B 成本 $3000 vs 模型 A $30000，省下的 $27000 足够覆盖 12% 失败处理成本还有大量盈余。

示范回答（追问）：

追问 1（总成本计算）：模型 A / 每百万次：$30000 + (5%×0.1×$2/人工) ≈ $31000。模型 B / 每百万次：$3000 + (12%×30% 重试成本 + 5% 转人工×$2) ≈ $9000。具体数字取决于业务但 10x 差距通常难靠准确率弥补。

追问 2（routing 架构）：speculative routing——模型 B 输出 tool_call 时看 logprob，低于置信度阈值自动升级到模型 A。或轻量分类器预判难度，简单走 B 复杂走 A。本质是"便宜的试，没把握再上贵的"。

追问 3（微调数据）：三个来源——1）大模型（GPT-5）的正确 tool call 做 teacher 蒸馏；2）生产日志中采集真实请求 + 正确路径，人工审核后入训练集；3）针对性构造对抗样本，把已知失败 case 加入。关键是负样本质量 > 正样本数量。

追问 4（BFCL gap）：BFCL v3 测的是给定 prompt + 工具定义，模型能否正确选择工具并填参数。局限：1）单轮为主（真实多是多轮）；2）工具简单（真实有嵌套参数和条件必填）；3）没有延迟约束（生产 800ms vs 200ms 决定 UX）；4）没有对抗样本（无人故意 prompt injection）；5）没有成本权重（只看准确率不看成本）。BFCL 分数 ≠ 生产表现，需要自己的 eval set。

详细解析：成本意识是区分 senior 和 junior 的关键。面试官想看候选人是否会算总成本而非只看单价、是否理解不同场景的容忍度差异、是否知道 routing 架构、是否有批判性思维质疑标准化 benchmark。字节和阿里都问过类似问题。

常见错误：
- 只看单价不算总成本
- 一刀切选模型不区分场景
- 把 BFCL 分数当生产表现——不了解 benchmark 局限
- routing 架构不提置信度阈值

主要依据：BFCL v3 Leaderboard 2026（评测框架和模型排名）[来源: Berkeley Function Calling Leaderboard]；Speculative Routing 实践（置信度阈值 routing）[来源: Anthropic]；模型成本对比分析 2026 [来源: FutureAGI]。

---

## 来源

- [OpenAI Function Calling Guide](https://platform.openai.com/docs/guides/function-calling)
- [Anthropic Tool Use Documentation](https://docs.anthropic.com/en/docs/build-with-claude/tool-use)
- [Evaluating LLM Tool Use in 2026: The Four-Step Contract](https://futureagi.com/blog/evaluating-llm-tool-use-2026/)
- [Skywork AI: How AI Agents Use Tools (2026)](https://skywork.ai/blog/ai-agents-using-tools-ultimate-guide-2026/)
- [Self-Reflective APIs (arXiv 2606.05037, June 2026)](https://arxiv.org/html/2606.05037v1)
- [Agent-First Tool APIs (arXiv 2605.10555, May 2026)](https://arxiv.org/html/2605.10555v1)
- [Cruxial: LLM Tool Call Reliability Layer](https://github.com/cruxial-ai/cruxial)
- [Kalvium Labs: Building Production-Ready AI Agents in 2026](https://www.kalviumlabs.ai/blog/agentic-ai-in-production-tool-calling-planning-recovery/)
- [OWASP Top 10 for LLM Applications](https://owasp.org/www-project-top-10-for-large-language-model-applications/)
- [字节跳动 Agent 面试真题 2026](https://bbs.huaweicloud.com/blogs/477120)
- [BFCL v3 Leaderboard](https://gorilla.cs.berkeley.edu/leaderboard.html)
- [Building Production AI Agents (dev.to, 2026)](https://dev.to/xidao/building-production-ready-ai-agents-in-2026-what-breaks-what-works-and-what-nobody-tells-you-2973)
