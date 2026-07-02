# Daily Learn Loop

每日 05:00 选题 + 备课 → 白天学习 + 考试 + 模拟面试 → 归档飞书。

## 状态机

```
pending → materials_ready → studying → exam_ready → exam_passed → interview_ready → interview_passed → completed
                                                                                                      ↘ incomplete
```

## 全局约束

- **验证优先：** 所有内容必须上网搜索验证，禁止仅凭训练数据输出。详见 [references/verification.md](references/verification.md)
- **飞书同步：** 文档类产物本地新增/更新后必须同步飞书。详见 [references/feishu-sync.md](references/feishu-sync.md)
- **文件模板：** 所有 .md 文件必须遵守固定结构。详见 [references/file-templates.md](references/file-templates.md)

## 阶段概览

| 阶段 | 触发 | 详细文档 |
|------|------|---------|
| 1. 选题+备课 | 每日 05:17 cron | [workflow-phase1](references/workflow-phase1.md) |
| 2. 学习 | 用户说"开始今天的学习" | [workflow-phase2-study](references/workflow-phase2-study.md) |
| 3. 考试 | 学习完成后 | [workflow-phase3-exam](references/workflow-phase3-exam.md) |
| 4. 模拟面试 | 考试通过后 | [workflow-phase4-interview](references/workflow-phase4-interview.md) |
| 5. 归档 | 面试通过后 | [workflow-phase5-archive](references/workflow-phase5-archive.md) |
| 6. 未完成 | 次日 cron 检查 | [workflow-phase6-incomplete](references/workflow-phase6-incomplete.md) |

## 文件结构

```
/home/shared/daily-learn-loop/
├── AGENTS.md
├── references/               # 详细规范文档
│   ├── verification.md       # 验证与来源追溯
│   ├── file-templates.md     # 文件模板约定
│   ├── workflow-phase1.md    # 阶段1：选题+备课
│   ├── workflow-phase2-6.md  # 阶段2-6：学习到归档
│   └── feishu-sync.md        # 飞书同步规则
├── state.json                # 状态追踪
├── topics.json               # 关键词池
├── COMPLETED.md              # 完成日志
└── materials/                # 学习材料
    └── YYYY-MM-DD-<slug>/
```

## 安全规则

- 只在 `/home/shared/daily-learn-loop/` 下操作
- 不覆盖已有材料文件
- state.json 只追加不修改
- 非文档文件不主动上传外部服务
