# 验证与来源追溯

**所有内容必须经过互联网搜索验证，禁止仅凭训练数据输出。**

## 验证工具

- **WebSearch**：搜索论文、官方文档、技术博客、面经
- **WebFetch**：深入阅读具体页面
- **browser-act**：JS 渲染页面、需登录的网站
- **Agent (Explore)**：大规模多源搜索
- **agent-reach skill**：多平台（GitHub/小红书/B站/Reddit/Twitter/网页），面经搜索优先用它搜国内社区和 GitHub 仓库

## 来源追溯

- 关键结论必须用 `[来源: 名称]` 标注出处
- 文末 `## 来源` 列出完整 URL，survey 论文列最前
- 综合推理需写明"综合自哪些来源"；无外部验证需标注"仅为推理"
- 考题和面试题必须标注题型依据（面经/官方文档/公开实践）

## 各阶段验证要求

| 阶段 | 验证要求 |
|------|----------|
| **备课** | 知识点 ≥2 独立来源；必须用 agent-reach 多源搜索；必须引用 ≥1 篇 survey |
| **学习** | 用户提问先搜索再回答；资料冲突以最新为准并修正教案 |
| **考试** | 出题前用 agent-reach 搜小红书+GitHub 面经；答案对照官方文档 |
| **面试** | 验证题目符合当前面试趋势；参考答案搜索社区最佳思路 |

## 验证失败处理

- 无可靠来源 → 告知用户"未找到外部验证，仅为推理"
- 与教案矛盾 → 更新教案，标注日期和来源
- 面试题更好 → 替换或补充进 interview.md

## agent-reach 搜索流程（备课/出题必须）

**Step 1: agent-reach 多源搜索**
- 小红书：`<关键词> 面试`、`<关键词> 面经`
- GitHub：`interview questions <keyword>`、`awesome <keyword>`
- B站：`<关键词> 教程`、面试分享
- 网页：面经文章

**Step 2: WebSearch 补充**
- arxiv.org survey 论文
- 官方文档（OpenAI/Anthropic/LangChain 等）
- 技术博客

**Step 3: WebFetch 深入阅读**

**Fallback：** agent-reach skill 不可用时回退 MCP 直调 xiaohongshu-mcp（localhost:18060）

## 必查 Survey 论文

Agent/LLM 类主题默认必查：
- Tool Learning with LLMs (arXiv 2405.17935)
- AI Agent Systems (arXiv 2601.01743)
- Agentic Design Patterns (arXiv 2601.19752)
- Architectural Design Decisions (arXiv 2604.18071)
- LLM Agent Evaluation Survey (arXiv 2503.16416)
