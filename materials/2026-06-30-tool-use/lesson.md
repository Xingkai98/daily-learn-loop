# Tool Use / Function Calling — 教案

## 1. 概念定义

**Tool Use（工具使用）**，在 LLM 领域通常称为 **Function Calling**，指的是让大语言模型在生成文本的过程中，能够调用外部工具（API、数据库、代码执行器等）来获取信息或执行操作。这是 LLM 从"对话机器"进化为"自治 Agent"的关键能力。

### 直觉类比

想象一个实习生。你问他"今天天气怎么样"，他不能凭记忆回答，因为他不知道当前的天气。但他可以：
1. 意识到自己不知道 → 这是**意图识别**
2. 打开天气 App 查一下 → 这是**工具调用**
3. 把结果用自己的话汇报给你 → 这是**结果整合**

Tool Use 就是给 LLM 装上"手"，让它不只是动嘴。

### 关键术语

| 术语 | 含义 |
|------|------|
| Tool/Function Schema | 用 JSON Schema 描述一个工具的接口：名称、参数、返回值 |
| Tool Call | 模型输出的结构化指令，指定要调用哪个工具、传什么参数 |
| Tool Response | 工具返回的结果，作为下一步对话上下文的一部分 |
| Orchestration | 协调多次工具调用的流程（顺序、并行、条件） |
| Grounding | 通过工具将模型输出与外部事实对齐 |

## 2. 核心原理

### 2.1 工作原理

Tool Use 的核心是一个"暂停-调用-恢复"的循环：

```
User: "帮我查一下北京今天天气，如果下雨就提醒我带伞"
  ↓
LLM: 思考 → 决定调用 get_weather(city="北京")
  ↓ (暂停生成)
System: 执行 get_weather → 返回 {"weather": "晴", "temp": 35}
  ↓ (恢复生成)
LLM: "北京今天晴天，温度 35°C，不需要带伞，但注意防晒！"
```

### 2.2 Function Calling 的工程实现

以 OpenAI 的实现为例：

**Step 1: 定义工具**
```json
{
  "name": "get_weather",
  "description": "获取指定城市的当前天气",
  "parameters": {
    "type": "object",
    "properties": {
      "city": {"type": "string", "description": "城市名称"}
    },
    "required": ["city"]
  }
}
```

**Step 2: 模型推理**
模型接收到用户消息 + 工具列表后，可以输出两种 token：
- **普通文本 token**：正常的文本回复
- **特殊 token**（如 `<tool_call>`）：表示要调用工具的结构化数据

**Step 3: 工具执行**
框架解析模型输出的 tool call，执行对应的函数，得到结果。

**Step 4: 结果回传**
将 tool_call 和 tool_response 作为新消息追加到对话历史中，模型基于新信息继续生成。

### 2.3 训练方法

模型不是天生就会 Tool Use 的，需要专门的训练：

**方法 A：Fine-tuning with tool-calling data**
- 构造包含 tool call 的对话数据集
- 每条数据包含：用户意图 → 模型决定调用工具 → 工具返回 → 模型总结
- 数据集格式需要符合特定 schema（如 OpenAI 的 messages format）
- 代表性工作：Gorilla (UC Berkeley)，专门微调 LLaMA 用于 API 调用

**方法 B：Prompt-based instruction**
- 在 system prompt 中描述 tools 并给出示例（few-shot）
- 不需要微调，但可靠性较低
- 适用于通用模型如 GPT-4、Claude

**方法 C：RLHF with tool-use reward**
- 在 RLHF 阶段加入工具使用正确性的 reward signal
- 惩罚幻觉式的工具调用和参数错误

## 3. 关键细节

### 3.1 Tool Schema 设计

好的 schema 设计直接影响调用成功率：

- **name 要精确**：`search_users` 比 `search` 好，避免歧义
- **description 要充分**：描述什么时候该用这个工具、参数的含义和边界
- **参数类型要严格**：string/number/boolean/enum，枚举值要列全
- **required 字段要准确**：如果不标记 required，模型可能不传导致调用失败

### 3.2 错误处理

Tool Use 有几种常见失败模式：

| 失败模式 | 现象 | 缓解方案 |
|----------|------|----------|
| 幻觉调用 | 调用不存在的工具 | Schema 中明确列出可用工具 |
| 参数错误 | 类型错误、缺必填 | Retry with error message |
| 过度调用 | 频繁调用简单工具 | 设置 max_turns |
| 调用不足 | 该问工具却自己编 | 在 prompt 中强调"不知道就查" |
| 死循环 | 不断重试同一调用 | Circuit breaker + max retries |

### 3.3 并行 vs 串行调用

模型可以在一个 turn 中发起**多个独立的 tool call**：

```
User: "比较一下北京和上海的天气"
  ↓
LLM 输出两个 tool_call（可并行执行）:
  - get_weather(city="北京")
  - get_weather(city="上海")
  ↓ (并行执行)
System: [北京结果, 上海结果]
  ↓
LLM: "北京 35°C 晴天，上海 30°C 小雨..."
```

并行调用能显著降低延迟，但需要框架支持。

### 3.4 Structured Output

Tool Use 的一种延伸是 **Structured Output**——要求模型输出符合特定 JSON Schema。这与 Tool Use 共用底层能力（模型学会输出结构化数据），但用途不同：
- Tool Use：触发外部行为
- Structured Output：约束输出格式（如提取字段、分类标签）

## 4. 实际应用

| 场景 | 工具类型 | 示例 |
|------|----------|------|
| 搜索增强 | Web Search API | 回答时效性问题 |
| 数据处理 | SQL / DataFrame | "这个月销售额最高的产品" |
| 代码执行 | Python Sandbox | "计算这个积分的值" |
| 外部操作 | Email / Slack API | "帮我发一封邮件给张三" |
| 知识库查询 | Vector DB / RAG | "公司内部文档里关于 XX 的规定" |
| 多模态 | Image Gen / Vision API | "根据描述生成一张图" |

**典型框架：**
- OpenAI Function Calling / GPT Actions
- Anthropic Tool Use (Claude)
- LangChain Tools / Agents
- Google Gemini Function Calling

## 5. 常见误区

1. **"Tool Use 就是 RAG"**——不是。RAG 是检索文档内容，Tool Use 的范围更广（执行操作、调用 API、运行代码）。RAG 可以看作 Tool Use 的一个子集（工具是向量数据库）。

2. **"模型越多工具越好"**——不是。过多的工具会让模型选择困难（choice paralysis），增加错误调用概率。最佳实践是每个场景只暴露相关工具（5-10 个）。

3. **"Tool Use 是后加的功能所以不可靠"**——现在的 SOTA 模型（GPT-4, Claude 4）的 Tool Use 是训练阶段就深度融合的，可靠性已不低。但失败率仍然 > 0，需要工程兜底。

4. **"Function calling 不需要 prompt engineering"**——需要。Schema 的 description 就是 prompt engineering。描述的质量直接影响调用准确率。

5. **"Tool call 的结果直接给用户看就行"**——不应该。工具返回的原始数据（如 JSON）应该由模型重新组织成自然语言再输出。这既是 UX 要求，也是安全要求（避免泄露内部数据结构）。

## 6. 延伸阅读

## 来源

**Survey 论文：**
- **Tool Learning with Large Language Models: A Survey** (Qu et al., arXiv 2405.17935, 2024) — 四阶段分类法（task planning→tool selection→tool calling→response generation），发表在 *Frontiers of Computer Science* 2025。当前最全面的工具学习综述。[https://arxiv.org/abs/2405.17935]
- **LLM With Tools: A Survey** (Shen, arXiv 2409.18807, 2024) — 工具集成标准化范式，涵盖 fine-tuning 和 in-context learning 方法，以及 LLM 创建自身工具的探索。[https://arxiv.org/abs/2409.18807]
- **What Are Tools Anyway? A Survey from the Language Model Perspective** (Wang et al., arXiv 2403.15452, 2024) — 从 LLM 视角定义"工具"的概念边界和分类框架。[https://arxiv.org/abs/2403.15452]

**其他来源：**
- **OpenAI Function Calling Guide** — 官方最佳实践
- **Anthropic Tool Use Documentation** — Claude 的工具使用范式（包括 computer use）
- **Gorilla: Large Language Model Connected with Massive APIs** (Patil et al., 2023)
- **Toolformer: Language Models Can Teach Themselves to Use Tools** (Schick et al., 2023)
- **Berkeley Function Calling Leaderboard (BFCL)** — 评估模型 Function Calling 能力的基准
