# Daily Learn Loop

每日 05:00 自动选关键词 → 备课 → 白天用户学习+考试+模拟面试 → 归档。

## 状态机

```
pending → materials_ready → studying → exam_ready → exam_passed → interview_ready → interview_passed → completed
                                                                                                      ↘ incomplete (当天未完成)
```

## 全局约束：验证优先（适用于所有阶段）

**所有内容必须经过互联网搜索验证，禁止仅凭训练数据输出。**

### 可用的验证工具

- **WebSearch**：首选，搜索论文、官方文档、技术博客、面试面经
- **WebFetch**：获取具体页面内容，深入阅读
- **browser-act**：访问需要 JS 渲染的页面、需要登录的网站（如面经论坛）
- **Agent (Explore)**：大规模多源搜索验证

### 各阶段验证要求

| 阶段 | 验证要求 |
|------|----------|
| **备课** | 每个知识点至少查 2 个独立来源；教案中标注引用 |
| **学习** | 用户提问时，先搜索再回答；如发现教案内容与最新资料冲突，以最新资料为准并**当场修正教案** |
| **考试** | 出题前搜索当月/当年的面试面经，确认题目是大厂确实在问的；答案需对照官方文档验证 |
| **面试** | 面试题需验证是否符合当前面试趋势；参考答案需搜索社区最佳回答思路 |

### 验证失败处理

- 如果搜索不到可靠来源 → 明确告知用户"该结论仅基于推理，未找到外部验证"
- 搜索到与教案矛盾的资料 → 更新教案，标注更新日期和来源
- 搜索到的面试题比教案更好 → 替换或补充进 interview.md

---

## 阶段 1：选题 + 备课（05:00 自动执行）

### 1a. 选关键词

从 `topics.json` 的 `pool` 中选一个关键词。选择规则：
- 优先选 `status: pending` 且距今最久远的
- 避免 7 天内重复的主题（检查 `category` + `tags`）
- 如果所有 pending 都太近，从 `suggestions` 里随机选一个新的
- 选好后在 `topics.json` 中标记 `last_picked` 为当天日期

关键词可以是：
- AI/大模型：Transformer、Attention、RLHF、RAG、Fine-tuning、Prompt Engineering、LoRA、KV Cache、MoE 等
- Agent：Tool Use、Planning、Memory、Multi-Agent、ReAct、Function Calling 等
- 传统 CS：分布式一致性、数据库索引、网络协议、操作系统调度、编译原理等
- 工程实践：系统设计、性能优化、调试方法论等

### 1b. 备课 —— 搜索验证 + 生成全部材料

**备课流程（必须按顺序）：**

1. **先搜索**：用 WebSearch 搜索关键词的权威资料
   - 搜索官方文档 / 经典论文（如 arxiv.org）
   - 搜索技术博客 / 教程（如 distill.pub, lilianweng.github.io, jaykmody.com 等）
   - 搜索面试面经（搜索"<关键词> 面试题"、"<关键词> 大厂面试"、"<keyword> interview questions"）
   - 如页面需 JS 渲染或登录，用 browser-act 访问
   - 记录至少 3 个可靠来源

2. **再生成**：基于验证后的资料，在 `materials/YYYY-MM-DD-<slug>/` 下生成以下文件

3. **最后标注**：每个文件末尾附上来源列表

**文件清单：**

1. **lesson.md** — 教案/学习材料
   - 概念定义、核心原理
   - 关键算法/机制（含伪代码或图解描述）
   - 实际应用场景
   - 常见误区
   - 延伸阅读方向
   - **末尾：来源列表**

2. **exam.md** — 考题（不含答案）
   - 5-8 道题，覆盖：概念理解、原理推导、实际应用、边界 case
   - 题型：选择题 + 简答题 + 场景题
   - 每道题标注难度（⭐/⭐⭐/⭐⭐⭐）
   - **必须验证**：搜索近期面经确认这些题是大厂确实在问的

3. **exam-answers.md** — 考题答案
   - 每题详细解析，不只给答案还要解释为什么
   - **答案需对照官方文档/权威来源验证**

4. **interview.md** — 模拟面试题（不含答案）
   - 5-8 道面试题
   - 对标互联网大厂 / AI 独角兽面试难度
   - 包含：基础知识深挖 + 系统设计 + 场景分析 + 追问链
   - **必须验证**：搜索"<关键词> senior engineer interview questions" 确认题型

5. **interview-answers.md** — 模拟面试参考答案
   - 每题给出评分要点和满分回答示范
   - 列出常见错误回答及纠正
   - **末尾：来源列表**

### 1c. 更新状态

```json
{
  "date": "YYYY-MM-DD",
  "slug": "...",
  "keyword": "...",
  "category": "...",
  "status": "materials_ready",
  "materials_path": "materials/.../",
  "sources_verified": true,
  "sources_count": N,
  "created_at": "ISO-8601"
}
```

### 1d. 通知用户

输出当天简报，包含关键词、材料路径、题数、来源数量。

---

## 阶段 2：学习（用户手动触发）

用户说"开始今天的学习"或等效指令时：

1. 读取当天 `lesson.md`
2. 以教学方式呈现内容（分段讲解，关键概念加粗，复杂原理用类比）
3. 每讲完一个子主题，主动问用户是否有疑问
4. **用户提问时，必须先 WebSearch 验证再回答**（见全局约束）
5. 讲完后问用户是否准备好考试

---

## 阶段 3：考试（学习完成后触发）

1. 读取 `exam.md`，逐题呈现
2. 用户作答后，对照 `exam-answers.md` 评判
3. 答错 → 指出错误，**搜索补充资料**重新讲解，让用户再答
4. 直到全部答对 → 状态更新为 `exam_passed`
5. **考后验证**：用 WebSearch 搜索最新面经，如果发现教案的题与实际面试题差距大，更新 exam.md

---

## 阶段 4：模拟面试（考试通过后触发）

1. 读取 `interview.md`
2. **面试前验证**：用 WebSearch 搜索近期大厂面试真题，确认 interview.md 的题还在考
3. 模拟真实面试场景：
   - 你扮演面试官，用户扮演候选人
   - 逐题提问，每道题包含追问链
   - 面试风格对标大厂（Google/Meta/ByteDance/OpenAI/Anthropic）
4. 每题回答完毕后：
   - **先搜索验证**：确认你的评判标准符合行业共识
   - 给出评价（优点 + 可改进点）
   - 如果回答不够好，给一次修正机会
   - 直到和用户达成共识认为回答合格

**面试难度标准：**
- L1：基础概念清晰度
- L2：原理深度（能推导、能类比、能区分相似概念）
- L3：实战经验（遇到过什么坑、怎么解决的）
- L4：系统思考（这个技术在整个系统中的位置、trade-off）

5. 全部通过后 → 状态更新为 `interview_passed`

---

## 阶段 5：归档（面试通过后触发）

### 5a. 生成面试实录

在材料目录下追加 `interview-transcript.md`：
- 每题记录：用户的原始回答 + 点评（哪些好、哪些不足）+ 追问过程
- 文末附总体评价（优点汇总 + 待加强清单）

### 5b. 更新本地追踪

1. 更新 `COMPLETED.md`
2. 更新 `state.json` 状态为 `completed`
3. 更新 `topics.json` 状态为 `completed`

### 5c. 上传飞书云文档

使用 lark-drive skill 将全部材料上传到飞书云空间：
1. 创建文件夹：`Daily-Learn/YYYY-MM-DD-<slug>/`
2. **上传前检查**：验证所有 .md 文件的 markdown 格式——确保代码块有闭合的 \`\`\`、没有孤立的 4 空格缩进、成对的 \`\` 内联代码标记
3. 上传以下文件为 docx：
   - `lesson.md` → `教案-<keyword>.docx`
   - `exam.md` → `考题-<keyword>.docx`
   - `exam-answers.md` → `考题答案-<keyword>.docx`
   - `interview.md` → `面试题-<keyword>.docx`
   - `interview-answers.md` → `面试参考答案-<keyword>.docx`
   - `interview-transcript.md` → `面试实录-<keyword>.docx`
3. 上传成功后，在 `state.json` 中记录飞书文件夹链接

---

## 阶段 6：未完成处理

如果当天 23:59 前用户未完成全部流程：
- Cron 在次日选题时检查前一天状态
- 如果仍是 `materials_ready` / `studying` / `exam_ready` / `exam_passed` / `interview_ready` → 标记为 `incomplete`
- 未完成的关键词可以留到以后再学

---

## 文件结构

```
/home/shared/daily-learn-loop/
├── CLAUDE.md                  # 本文件
├── state.json                 # 状态追踪（机器可读）
├── topics.json                # 关键词池 + 选择记录
├── COMPLETED.md               # 完成日志（人类可读）
└── materials/
    ├── YYYY-MM-DD-<slug>/
    │   ├── lesson.md
    │   ├── exam.md
    │   ├── exam-answers.md
    │   ├── interview.md
    │   └── interview-answers.md
    └── ...
```

## 安全规则

- 只在 `/home/shared/daily-learn-loop/` 下操作
- 不覆盖已有的材料文件
- state.json 只追加不修改已有条目
- 生成本地文件，不上传外部服务（除非用户要求）

## 飞书归档规则

- 每次学习完成后，必须使用 lark-drive skill 将所有材料上传到飞书
- 目标路径：`Daily-Learn/YYYY-MM-DD-<slug>/`
- 全部 6 个文件以 docx 格式导入
- 如果 `Daily-Learn` 文件夹还不存在，先创建
