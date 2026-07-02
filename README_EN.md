# Daily Learn Loop

A structured daily AI learning system — automated prep → exam → mock interview → archive.

## What is this

A structured learning workflow for AI, Agent, and CS topics. Every day at 5 AM, the system picks a keyword, researches it through web search, and generates a lesson plan, exam questions, and mock interview questions. During the day, you study, take the exam, and do a mock interview. Once all passed, everything is archived.

## Directory Structure

```
├── AGENTS.md                       # AI agent workflow spec (entry point)
├── references/                     # Detailed spec documents
│   ├── verification.md             #   Verification & source tracing
│   ├── file-templates.md           #   File template conventions
│   ├── workflow-phase*.md          #   Phase-by-phase workflows
│   └── feishu-sync.md              #   Feishu sync rules
├── topics.example.json             # Topic pool (example)
├── state.example.json              # State tracking (example)
├── COMPLETED.example.md            # Completion log (example)
└── materials/                      # Learning materials archive
    └── YYYY-MM-DD-<slug>/
```

## Workflow

```
Select topic → Prep (research + generate materials)
  → Study (lesson uploaded, read at once)
  → Exam (one question at a time, must pass all)
  → Mock interview (follow-up chains, Big Tech difficulty)
  → Archive (all materials synced)
```

## Keyword Coverage

- **AI/LLM:** Transformer, Attention, RLHF, RAG, Fine-tuning, Prompt Engineering, LoRA, KV Cache, MoE, Diffusion, etc.
- **Agent:** Tool Use, Planning, Memory, Multi-Agent, ReAct, MCP, Architecture, Security, etc.
- **CS Fundamentals:** Distributed consensus, database indexing, network protocols, OS scheduling, compiler design, etc.
- **Engineering:** System design, performance optimization, cost engineering, etc.

## Quick Start

1. `cp topics.example.json topics.json`
2. `cp state.example.json state.json`
3. `cp COMPLETED.example.md COMPLETED.md`
4. Open the project with Claude Code and say "initialize the daily learn system"
5. Or manually set up a cron: `CronCreate` to trigger daily at 05:17

## Material Format

Each keyword generates 6 files:

| File | Purpose |
|------|---------|
| `lesson.md` | Teaching material with concepts, principles, pitfalls, sources |
| `exam.md` | Exam questions (multiple choice + short answer + scenario) |
| `exam-answers.md` | Answers with detailed explanations |
| `interview.md` | Mock interview questions with follow-up chains |
| `interview-answers.md` | Model answers, scoring criteria, common mistakes |
| `interview-transcript.md` | Interview transcript |

All content is web-search-verified with source citations. Questions are calibrated to Big Tech / AI unicorn interview standards (Google, ByteDance, OpenAI, Anthropic).

## Core Rules

- All content must be verified through web search — no training-data-only answers
- Must cite at least 1 survey paper per topic
- Interview questions benchmarked against top-tier companies

## License

MIT
