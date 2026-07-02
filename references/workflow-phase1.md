# 阶段 1：选题 + 备课（05:00 自动执行）

## 1a. 选关键词

从 `topics.json` 的 `pool` 中选一个关键词：
- 优先选 `status: pending` 且距今最久远的
- 避免 7 天内重复主题
- 所有 pending 都太近时从 `suggestions` 随机选
- 更新 `last_picked` 为当天日期

## 1b. 备课流程

**必须按顺序：**

1. **先搜索**：按 `references/verification.md` 中的 agent-reach 搜索流程执行多源搜索
   - ≥3 可靠来源（含 ≥1 篇 survey）
   - 建立"内容点 → 来源"映射
2. **再生成**：在 `materials/YYYY-MM-DD-<slug>/` 下生成 5 个文件
3. **最后标注**：正文标注来源，末尾列完整来源（survey 最前）

文件清单与模板见 `references/file-templates.md`。

## 1c. 更新状态

```json
{
  "date": "YYYY-MM-DD", "slug": "...", "keyword": "...",
  "category": "...", "status": "materials_ready",
  "materials_path": "materials/.../",
  "sources_verified": true, "sources_count": N, "created_at": "ISO-8601"
}
```

## 1d. 通知用户

输出简报：关键词、类别、路径、题数、面试题数、状态。
