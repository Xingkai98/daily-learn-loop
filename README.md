# Daily Learn Loop

每日 AI 关键词学习系统 — 自动备课 → 考试 → 模拟面试 → 归档。

## 这是什么

一个结构化的 AI/Agent/CS 知识学习工作流。每天凌晨自动选一个关键词，搜索验证后生成教案、考题和面试题。白天你学习、考试、模拟面试。全部通过后材料归档到飞书。

## 目录结构

```
├── AGENTS.md                       # AI Agent 工作流规范（入口）
├── references/                     # 详细规范文档
│   ├── verification.md             #   验证与来源追溯
│   ├── file-templates.md           #   文件模板约定
│   ├── workflow-phase*.md          #   各阶段流程
│   └── feishu-sync.md              #   飞书同步规则
├── topics.example.json             # 关键词池（示例）
├── state.example.json              # 状态追踪（示例）
├── COMPLETED.example.md            # 完成日志（示例）
└── materials/                      # 学习材料归档
    └── YYYY-MM-DD-<slug>/
```

## 工作流

```
选题 → 备课(搜索验证+生成材料)
  → 学习(教案上传飞书,一次性阅读)
  → 考试(逐题作答,全对才过)
  → 模拟面试(追问链,对标大厂)
  → 归档(全部材料上传飞书)
```

## 关键词覆盖

- **AI/LLM：** Transformer、Attention、RLHF、RAG、Fine-tuning、Prompt Engineering、LoRA、KV Cache、MoE、Diffusion 等
- **Agent：** Tool Use、Planning、Memory、Multi-Agent、ReAct、MCP、Agent Architecture、Security 等
- **CS 基础：** 分布式一致性、数据库索引、网络协议、操作系统调度、编译原理等
- **工程实践：** 系统设计、性能优化、成本控制等

## 快速开始

1. `cp topics.example.json topics.json`
2. `cp state.example.json state.json`
3. `cp COMPLETED.example.md COMPLETED.md`
4. 用 Claude Code 打开项目，说"初始化每日学习系统"
5. 或手动设置 cron：`CronCreate` 每天 05:17 触发备课流程

## 材料格式

每个关键词生成 6 个文件：

| 文件 | 用途 |
|------|------|
| `lesson.md` | 教案，含概念/原理/误区/来源 |
| `exam.md` | 考题（选择+简答+场景） |
| `exam-answers.md` | 答案+解析 |
| `interview.md` | 面试题+追问链 |
| `interview-answers.md` | 满分回答+评分要点+常见错误 |
| `interview-transcript.md` | 面试实录 |

所有内容经互联网搜索验证，标注来源。考题对标大厂面试真题。

## 核心约束

- 所有内容上网搜索验证，禁止仅凭训练数据输出
- 必须引用至少 1 篇 survey 论文
- 面试题对标 Google/ByteDance/OpenAI/Anthropic

## 许可

MIT
