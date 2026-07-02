# Agent Planning & Reasoning — Interview Reference Answers

> 评分标准依据：FutureAGI 2026 Architecture Survey 最佳实践、字节/阿里 Agent 面试评分要点、Oracle ReAct vs Plan-Execute Production Guide

---

## 1. ReAct 基础 (L1→L2)

**题目原文：** 请描述 ReAct 的完整执行循环。用一个具体的任务（比如帮用户退订单）演示 Thought → Action → Observation 的全过程。每个环节是模型生成的还是工程代码决定的？

**追问：**
- ReAct 和简单的 Chain-of-Thought 有什么区别？CoT 能做 Agent 吗？
- 如果某一步的 Observation 返回了完全出乎意料的结果（比如查订单返回了 500 错误），ReAct 有什么机制来应对？够不够？
- 用 ReAct 做一个 15 步的复杂任务时，最可能在哪一步出问题？为什么？

**满分回答要点：**
- 准确描述 Thought → Action → Observation 循环，并能区分 LLM 生成（Thought、Action 选择）和工程代码执行（Action 执行、Observation 返回）
- 明确指出 CoT 只能"想"不能"做"——没有工具调用就没有 Observation 反馈，形成不了闭环
- 对于 500 错误：ReAct 机制上是最适合应对的——Agent 可以在下一个 Thought 中分析错误并尝试替代方案。但"够不够"取决于是否有足够的 fallback 工具和错误分类
- 15 步任务最可能在 8-12 步出问题：上下文积累导致"漂移"——Agent 逐渐忘记原始目标，开始围绕次优路径打转

**示范回答：**

ReAct 的核心是一个紧密的 Thought → Action → Observation 循环。用退款场景：

> Thought: 用户要退款 #4521 订单，我需要先确认订单状态
> Action: search_orders("4521")  ← 模型决定调什么，工程代码执行
> Observation: {"order_id": "4521", "amount": 623, "status": "completed", "refundable": true}
> Thought: 订单有效且可退款，执行退款操作
> Action: refund_order("4521")
> Observation: {"refund_id": "RF-8891", "status": "success"}
> Thought: 退款成功，可以输出最终结果
> Final Answer: 已退款 ¥623，退款单号 RF-8891

各环节归属：Thought 由模型生成、Action 的"决定"由模型做出但执行是工程代码、Observation 是工具返回的原始数据。

**追问 1：ReAct 和 CoT 的核心区别**

ReAct 和 CoT 的核心区别：CoT 只有推理没有行动——相当于一个人坐在房间里想，没有手机、没有电脑、没有数据库。CoT 不能做 Agent，因为 Agent 的本质是"感知→决策→行动→反馈"的闭环，没有 Action 就没有反馈信号，无法应对需要外部信息或外部操作的任务。

**追问 2：500 错误的应对**

ReAct 天然适合处理异常——Agent 看到 500 错误后，可以在下一个 Thought 分析："订单查询返回 500，可能是服务问题，我先查询缓存或者换个方式确认"。但仅靠 ReAct 不够——需要配套的错误分类体系让模型知道这是可重试的临时错误。如果模型不理解 500 的含义，它可能把它当成"查询成功但结果为空"，这是 ReAct 的盲区。

**追问 3：15 步任务的脆弱点**

第 8-12 步是最危险的。上下文窗口累计了前 8 步的 Thought+Observation，Agent 容易"漂移"——忘记最初的目标是什么。比如本意是退订单，第 8 步 Agent 可能在研究为什么物流商换了承运商。

**常见错误：**
- 说"Thought 是 Prompt 写出来的"——其实是模型生成的
- 把 CoT 和 ReAct 混为一谈——CoT 没有 Action/Observation 环
- 认为 500 错误后 Agent 应该立即失败——ReAct 的优势就是能灵活应对异常

**详细解析：**
这道题考察的是对 Agent 最基础模式的理解深度。面试官不仅想知道你会背循环的四个步骤，更想确认你能区分"模型做的事"和"工程做的事"，以及你能预判 ReAct 在生产中的实际瓶颈。预期深度是 L2——需要展现生产级理解而非教科书复述。回答好的关键是把抽象循环落在一个具体的业务 trace 上，让面试官看到你真的跑过 Agent。

> 主要依据：Yao et al. ReAct 论文 (2022); kamacoder Agent Loop 专题

---

## 2. Plan-and-Execute (L2→L3)

**题目原文：** Plan-and-Execute 把"规划"和"执行"拆成两个独立的角色。这个设计解决了什么问题，又引入了什么新问题？

**追问：**
- Planner 需要多强的模型？Executor 呢？为什么通常用不同规格的模型？
- 执行到第 4 步时发现第 2 步的结果有问题，但第 3 步已经基于它执行完了。你的重规划机制怎么处理？
- Plan-and-Execute 的最大缺陷是什么？在什么场景下它完全不应该被使用？

**满分回答要点：**
- 解决：可审计性（蓝图可审）、成本优化（1 强+N 弱）、确定性（同样的任务同样的路径）
- 引入：规划-执行间的信息滞后、重规划复杂、强依赖规划质量
- 重规划不从头开始，从当前状态出发重排剩余步骤
- 最大缺陷：环境动态变化时蓝图过时

**示范回答：**

**解决的问题：**
1. **可审计性：** 蓝图在执行前可以被人工审阅，这在合规要求高的场景（金融、医疗）是关键能力
2. **成本优化：** Planner 用强模型（GPT-5/Claude Opus），Executor 用便宜模型（Haiku/4o-mini），N 步任务总成本 = 1×强 + N×便宜，对比 ReAct 的 N×强，长任务成本显著更低
3. **确定性：** 同样任务每次都走同样的路径，方便 A/B 测试和回归

**引入的问题：**
1. **规划滞后：** 蓝图基于执行前的环境状态，执行到第 4 步时环境可能变了，蓝图过时
2. **重规划复杂：** 需要判断"是该重试当前步、重做某几步、还是整个蓝图作废"

**追问 1：模型选择**

Planner 需要准确理解复杂目标并分解——这是模型能力的关键区分点，应该用最强模型。Executor 只需遵循简单指令（"用这些参数调这个工具"）——便宜模型足够。通常可以差一个 tier：Planner=Opus/Sonnet，Executor=Haiku。

**追问 2：重规划处理**

第 2 步的结果有问题，第 3、4 步已经基于它执行了：
> 首先判断第 2 步的错误是否影响第 3、4 步的输出。用依赖图找受影响的步骤。
> 如果影响：从第 2 步开始重做（不是从头！），然后执行修正后的第 3、4 步
> 如果第 3、4 步与新修正的第 2 步结果不冲突：只重新执行第 2 步，保留 3、4 的结果

**追问 3：完全不适用场景**

路径完全不可预测的任务——比如"帮我排查为什么这个服务延迟突然升高了"。这种需要看一个 metric 再决定查哪个 log、再决定看哪个 config 的任务，每一步都依赖上一步的发现，预先规划毫无意义，Plan-and-Execute 在这里比 ReAct 差。

**常见错误：**
- 重规划从头开始——正确做法是从当前状态出发
- 说 Planner 和 Executor 可以通用——浪费钱且没必要
- 不分析适用边界——Plan-and-Execute 不是银弹

**详细解析：**
Plan-and-Execute 是 Agent 架构从"能用"到"好用"的关键升级。面试官想听到的是你对 trade-off 的理解深度——不是简单说"省成本"，而是能说清楚规划滞后、重规划复杂度这些生产中的真实痛点。预期深度是 L3——你应该展现出部署过这套架构的人才有的经验，包括什么场景下不该用它。回答好的标准是每个优点和缺点都有具体的生产场景支撑，而不是泛泛而谈。

> 主要依据：InfoQ Plan-and-Execute 专题 (2026); Oracle Integration Blog; FutureAGI 2026

---

## 3. 模式选型 (L3)

**题目原文：** 三个任务场景选型，讲决策和 Trade-off。

**追问：**
- 任务 A 如果用 Plan-and-Execute 会有什么问题？
- 任务 B 如果用纯 ReAct 呢？成本的账怎么算？
- 在实际团队中，你怎么说服同事接受你的选型？

**满分回答要点：**
- 场景 A 选 ReAct：路径不确定，每一步依赖上一步
- 场景 B 选 Plan-and-Execute：步骤固定，可审计，成本优化
- 场景 C 选 Reflection：有客观评价标准（测试结果）
- 每个选择必须讲 Trade-off

**示范回答：**

**任务 A（换货）：选 ReAct。**
路径不确定——先查订单→看尺码信息→确认库存→决定是换还是退→执行。每一步的输出决定下一步，Plan-and-Execute 在这里会很难规划，因为"如果尺码没货"和"如果尺码有货"的后续步骤完全不同。硬用 Plan-and-Execute 的话，Planner 不得不枚举所有分支，计划会比 ReAct 的 Token 消耗还大。

**任务 B（季度报告）：选 Plan-and-Execute + ReAct 叶子执行。**
5 个数据源 + 固定步骤 + 需要合规审计。这里用纯 ReAct 的话：假设 12 步，每步用 Sonnet 级模型，12×强 = ~$0.36。Plan-and-Execute：1×Opus 规划 + 12×Haiku 执行 ≈ $0.08。成本差 4.5 倍。更重要的是，Plan-and-Execute 的蓝图可以在拉数据之前发给审计方确认，纯 ReAct 做不到——你只能事后追溯。

**任务 C（Race Condition Bug）：选 Reflection。**
有明确的客观评价标准（死锁复现率、测试 suite）。Reflection 每一次失败后产出的"教训"可以指导下次尝试不同的锁策略或操作顺序。ReAct 也能做，但没有"从失败中学习"的机制——每次尝试都是独立的。

**追问 1：任务 A 用 Plan-and-Execute 的问题**

如任务 A 分析中所述，Plan-and-Execute 在路径高度不确定的场景下会"过度规划"——Planner 不得不枚举所有分支路径（如果库存有货就换，没货就退，尺码不对要推荐替代品），最终生成的蓝图比 ReAct 的逐步决策消耗更多 Token，且执行到一半蓝图可能已经过时。

**追问 2：任务 B 用纯 ReAct 的成本账**

如任务 B 分析中所述，纯 ReAct 在 12 步任务中每步都需要强模型参与决策，总成本约 $0.36，而 Plan-and-Execute 混用强/弱模型只需约 $0.08，成本差约 4.5 倍。对于日均数千次调用的生产系统，这个差距意味着每月数千美元的差额。此外纯 ReAct 还丧失了蓝图的可审计性。

**追问 3：说服同事**

我会用一个 10 分钟的 brown-bag session：同样一个任务，让团队看两个 agent 的实际执行 trace。通常 ReAct 的 trace 是 35 行蜿蜒的日志，Plan-and-Execute 的是 3 行蓝图 + 清晰的执行步骤。眼见为实比任何理论都有说服力。然后算 Token 成本——在月报场景下，Plan-and-Execute 省 75% 成本，数字最有说服力。

**常见错误：**
- 三个场景都选 ReAct——不理解 Plan-and-Execute 的价值
- 不讲成本计算——缺乏工程意识
- 不讲 trade-off——显得没有深入思考

**详细解析：**
这是判断工程判断力的题。面试官不期待有唯一正确答案——他要看你的决策框架是否成熟。你需要在回答中同时展现：能算清楚成本账（工程意识）、能分析适用边界（架构功底）、能说服团队（沟通能力）。预期深度是 L3+——高级工程师应该能独立做架构选型并让团队接受。回答好的标准是每个选择后面都跟着具体的 trade-off 分析和量化数据，而不是只说"我觉得这个好"。

> 主要依据：kamacoder Agent Loop 面试专题; FutureAGI 2026; Oracle ReAct vs Plan-Execute

---

## 4. Reflection 的信任问题 (L2→L3)

**题目原文：** 你怎么知道 Agent 的自我反思是真的进步了，而不是在自我安慰？

**追问：**
- 如果 Agent 的第一次尝试就错了（比如把金额 540000 分写成了 "$54.00"），Reflection 能发现吗？什么情况下能、什么情况下不能？
- 你在生产环境中怎么给 Reflection 加上"外部锚点"？举一个具体例子。
- 如果 Critic 反馈本身就是错的（说改 A，其实应该改 B），Agent 会不会越改越坏？怎么防？

**满分回答要点：**
- 不用 Agent 的主观评价验证反思效果——用客观指标
- 区分"有外部验证"和"无外部验证"的任务
- 外部锚点减少"幻觉收敛"风险

**示范回答：**

**核心方法论：不要信 Agent 自己说的。**

Agent 说"我改好了"不是验证——它可能改得更错了但更"自信"了。验证反思效果的三个层级：
1. **有标准答案的任务**（代码修复、SQL 生成）：看测试通过率。Reflection 版本 vs 无 Reflection 版本，pass@k 提升多少
2. **有外部数据的任务**（客服回答）：将 Reflexion 前后的回答 diff，看引用的外部数据是否更准确了。设一个 golden dataset 做人工标注对比
3. **开放型任务**（写作、翻译）：人工 pairwise 评分，Reflection 版本是否被更频繁地选为更好的版本

**追问 1：金额错误能否被发现**

金额错误（540000分→$54.00）能否被发现：
- **能发现的情况：** 如果上下文中有订单总金额的原始数据，Reflection 可以对比发现不一致
- **不能发现的情况：** 如果模型不知道"分"和"元"的转换规则，或者转换后数值巧合看起来合理（$5400 和 $54.00 量级不同可察觉但 $540.00 和 $54.00 模型容易判断错），Reflection 会确认错误答案

这就是"Internal Critic 的局限性"——Critic 和 Actor 是同一个模型，共享同样的知识盲区。

**追问 2：外部锚点设计**

业务系统场景（退款处理）——在 Reflection 步骤中注入一条规则检查：
> "退款金额是否匹配原始订单金额？如果金额不一致，无论 Agent 怎么说，都标记为需要人工审核。"

这种规则级别的锚点是绝对可靠的——不依赖模型判断。

**追问 3：越改越坏的防护**

当 Critic 反馈是错误的时候，Agent 确实可能越改越坏。三层防护：
1. **保留原始版本**：每次修改前存档，允许回退到任意历史版本
2. **渐进式修改**：每次只改一个维度，方便定位哪次修改引入了新错误
3. **主指标监控**：如果修改后的版本在核心指标（测试通过率、用户满意度）上反而下降，自动回退

**常见错误：**
- 用"Agent 自己觉得好了"作为验收标准——大忌
- 没有外部锚点验证就信任 Reflection 结果
- 不提越改越坏的风险——缺乏生产经验

**详细解析：**
信任/评估问题是 Agent 系统中最难也最容易被忽视的问题。面试官想看的是：你不天真——你知道模型会"自信地犯错"，而且你有系统性的方法来验证反思是否真的带来了进步。预期深度是 L3-L4——不仅要知道 Reflection 的原理，还要在生产中设计过验证机制。回答好的关键是把"不要信 Agent 自己说的"这个核心原则贯穿始终，并且用具体的规则锚点和回退机制来证明你真正落地过。

> 主要依据：Reflexion 论文 Shinn et al. 2023; 2026 大厂 Agent 面试全解析

---

## 5. 死循环与异常恢复 (L3→L4)

**题目原文：** "你的 Agent 调了三个工具就死循环了，异常处理在哪写的？"

**追问：**
- max_turns 怎么定的？太小误杀、太大浪费——你怎么做实验？
- 重复检测中"语义相同"怎么定义？
- "死循环"和"任务太难不会做"有什么区别？

**满分回答要点：**
- 四层防护体系
- 通过回放生产 trace 确定最优 max_turns
- 用工具名+参数 embedding 判断语义相同
- 死循环≈有进展但方向错/重复；任务太难≈无进展

**示范回答：**

**四层防护体系：**
1. max_turns=15（硬限制）
2. 重复检测：同一工具+参数连续 3 次 → 阻止，返回"你已尝试此操作多次，请换方案"
3. 无进展检测：3 轮无新信息 → 打断，强制输出当前已知并请求用户引导
4. 超时控制：全任务 120 秒上限

**追问 1：max_turns 实验**

回放 1000 条真实生产 trace，统计任务完成所需的步数分布。P95=11 步，P99=14 步。max_turns=15 覆盖了 99% 的正常任务。设 20 只增加了 1% 覆盖但多燃烧了大量失控 trace 的 Token。设 10 会误杀 5% 的正常任务——不可接受。

**追问 2：语义相同定义**

不做精确字符串匹配。用工具名 + 参数的 embedding 向量，cosine similarity > 0.95 判定为"语义相同"。这样 `"ORDER-123"` 和 `"order-123"` 以及 `"ORDER-000123"` 都会被识别为同一调用。

但也需要白名单——有些工具天然允许多次调用（如 `send_message`），这类工具在重复检测中设为豁免。

**追问 3：死循环 vs 太难**

| | 死循环 | 太难不会做 |
|---|---|---|
| Agent 表现 | 有进展/有动作但方向错 | 无进展/无新动作 |
| 工具调用 | 持续调用但参数微调 | 调用很少甚至不调用 |
| 上下文增长 | 每轮都有新 Thought | 上下文停滞 |
| 处理方式 | 打断+换方向 | 降级+请求人工 |

检测"太难"：连续 3 轮无工具调用或工具调用后无实质性新信息，判定为"任务超出能力"，降级到人工或返回"我目前无法完成"。

**常见错误：**
- max_turns 随便拍一个数字——应该基于 trace 数据
- 重复检测做精确字符串匹配——不实用
- 混淆死循环和"太难"——处理方式不同

**详细解析：**
异常处理和鲁棒性是区分 Demo Agent 和生产 Agent 的分水岭。面试官在探测你的运维成熟度——你是否系统性地思考过所有失败模式？还是只是拍脑袋设了个 max_turns？预期深度是 L3-L4——需要展现 SRE 级别的思维方式，每个参数都基于 trace 数据而非直觉。回答好的标准是四层防护体系层层递进且每层都有量化依据，同时能精准区分不同失败模式并给出不同的处理策略。

> 主要依据：华为云博客字节面试题; Baidu Developer Agent Error Handling Guide

---

## 6. 架构演进实战 (L4)

**题目原文：** "从 0 到 1 设计一个电商售后 Agent，最开始用纯 ReAct，上线后发现三个问题：高并发 Token 成本爆炸、复杂多步任务成功率低、出了问题难以追溯。你怎么一步步演进这个架构？"

**追问：**
- 你引入了 Plan-and-Execute 还是 Supervisor+Workers？为什么？
- 架构改变后，已有的 Prompt 和 Tool Schema 需要怎么改？
- 怎么衡量收益？用什么指标说服老板？
- 新一代 reasoning 模型会改变这些选择吗？

**满分回答要点：**
- 演进路径清晰且有数据驱动
- 不是为改而改，每个决定有量化依据
- 理解 reasoning 模型的趋势影响

**示范回答：**

**演进三阶段：**

**Phase 1 — 发现问题（运行 2 周后）：**
采集了以下数据：高峰期（晚 8-10 点）并发 500+ 请求，平均 Token 消耗 ¥0.42/请求，P95 延迟 8.3s，复杂多步任务（>5 步）成功率 72%，错误 trace 平均 37 步（大量在兜圈子），3 次线上事故都是"Agent 做了什么但不知道从哪查起"。

**Phase 2 — 引入 Plan-and-Execute（我选这个而不是 Supervisor+Workers）：**
选 Plan-and-Execute 而不是 Supervisor+Workers 的原因：
- 电商售后是流程化业务，任务步骤相对固定（查→审→退），而不是需要多领域专家
- Supervisor+Workers 适合"一个任务需要法律+财务+技术三个 Agent 交叉验证"的场景，这里是单一售后领域
- Plan-and-Execute 的审计能力是刚需——每次退款都要有可追溯的决策链

**Phase 3 — 生产优化：**
- 引入 Planner 缓存：80% 的任务类型是固定的（退换货、查物流、改地址），缓存常见规划模板，跳过 Planner 调用
- 引入模型路由：简单任务（单步查询）直接 Faiss 匹配后走规则，不走 Agent
- 升级 trace 系统：用 OpenTelemetry span 标记每个 step 的工具调用→结果→决策

**追问 1：为什么选 Plan-and-Execute 而不是 Supervisor+Workers**

如 Phase 2 分析中所述，电商售后是流程化业务，任务步骤相对固定（查→审→退），而不是需要多领域专家协同的场景。Supervisor+Workers 的强项是多领域 Agent 协作（如法律+财务+技术交叉验证），在单一售后领域反而增加了不必要的编排复杂度。此外 Plan-and-Execute 的审计能力是刚需——每次退款都要有可追溯的决策链，这在 Supervisor+Workers 的灵活委派模式下很难保证。

**追问 2：Prompt 和 Tool Schema 的改动**

- Planner prompt 增加了对每个 step 的输出格式约束（必须有明确的 success/failure 字段）
- Tool Schema 增加了 `step_id` 和 `trace_id` 元数据字段，让每个工具调用可追溯到蓝图步骤
- Executor 的 system prompt 从"你是一个 Agent"改为"你按计划执行单个步骤，不要自由发挥"
- 这是最大改动——限制 Executor 的自由度，原来 ReAct 的自由探索在 Executor 中成了危险

**追问 3：衡量收益与说服老板**

**收益（Phase 2 上线 1 个月后）：**
- Token 成本 ¥0.42 → ¥0.11/请求（降 74%）
- 复杂任务成功率 72% → 91%
- 事故定位时间 30min → 5min（因为 trace 从"长日志"变成了"蓝图+步骤"结构）
- 省下的月度成本 ¥12,000 用来说服老板

**追问 4：Reasoning 模型的影响**

新一代模型（Claude 4.7 extended thinking、GPT-5）模糊了 ReAct 和 Plan-and-Execute 的边界。模型可以在"内部思考"中隐式做规划，减少了对外部 Planner 的依赖。但我不认为这会淘汰 Plan-and-Execute——因为审计和控制的需求不取决于模型的内部能力。而且生产环境中，显式的、可追溯的蓝图在合规和调试上的价值是模型内部推理无法替代的。

**未来 2-3 年趋势：** 我认为架构会收敛到"reasoning 模型做隐式规划 + 轻量显式蓝图层（用于合规和控制）+ ReAct 叶子执行"。纯 ReAct 和纯 Plan-and-Execute 都会逐渐被混合架构取代。

**常见错误：**
- 只谈架构不谈数据——决策没有量化依据
- 改完架构不衡量收益——无法说服业务方
- 忽略 Prompt/Schema 的改动成本——只说架构不给实现细节

**详细解析：**
这是架构师级别的综合题。面试官想看的是端到端的系统思维：发现问题→设计方案→实施迁移→量化收益→展望未来，五步缺一不可。每个决策都必须有数据支撑，不能是"我觉得这样好"。预期深度是 L4——需要展现 Staff/Principal 工程师的系统演进能力，从具体数字（Token 成本、成功率、定位时间）到长期技术趋势判断都要覆盖。回答好的标准是让面试官觉得你可以独立负责一个 Agent 系统从 0 到 1 的架构演进。

> 主要依据：阿里 P7 架构师面经; FutureAGI LLM Agent Architectures 2026; Baidu Developer AI Agent Guide

---

## 来源

- [Yao et al., ReAct (2022)](https://arxiv.org/abs/2210.03629)
- [Shinn et al., Reflexion (2023)](https://arxiv.org/abs/2303.11366)
- [kamacoder Agent Loop 面试专题](https://notes.kamacoder.com/llm/app/react_reflection_planning.html)
- [FutureAGI: Agent Architecture Patterns 2026](https://futureagi.com/blog/agent-architecture-patterns-2026/)
- [FutureAGI: LLM Agent Architectures Core Components 2026](https://futureagi.com/blog/llm-agent-architectures-core-components/)
- [InfoQ Plan-and-Execute 专题 (2026)](https://xie.infoq.cn/article/159ae42cc328312c526a49cb7)
- [Oracle: ReAct vs Plan & Execute Guide](https://blogs.oracle.com/integration/react-vs-plan-execute-choosing-the-right-agent-thinking-pattern-in-oracle-integration)
- [2026 Agent面试终极攻略 (CSDN)](https://blog.csdn.net/2301_80239908/article/details/161602094)
- [Baidu Developer: AI Agent 面试全解析 (2026)](https://developer.baidu.com/article/detail.html?id=7670839)
- [Baidu Developer: AI Agent 框架设计全解析 (2026)](https://developer.baidu.com/article/detail.html?id=6794083)
