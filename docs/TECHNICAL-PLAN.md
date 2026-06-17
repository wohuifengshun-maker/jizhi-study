# 极致 · 智学私教 — 技术方案与开发蓝图

> 版本：v1.0 · 编写于 2026-06 · 面向开发团队
> 本文档是把演示原型（`prototypes/`）落地为正式产品的工程蓝图。

---

## 1. 产品目标与范围

### 1.1 一句话定义

一个绑定机构课程资料的 AI 自学网页系统，让留学生在课后通过向 AI 私教提问巩固知识点，配合进度追踪、学习足迹、薄弱复盘，最终达到通过课程考试的水平。即便零基础，也能借助系统自学通过课程。

### 1.2 与通用 AI 聊天的本质区别（即护城河）

| 能力 | 通用 ChatGPT | 本系统 |
|------|-------------|--------|
| 答案来源 | 模型通用知识 | **本校本课的课件、真题、考点**（RAG） |
| 教学方式 | 有问必答 | 苏格拉底式、检验理解、对标考试 |
| 进度 | 无 | 考点掌握地图 + 考试就绪度 |
| 记忆 | 单会话 | 跨会话持久：足迹、薄弱、进度 |
| 规划 | 无 | 对齐作业 due 的每日学习计划 |

护城河来自第一行和最后三行——这些都需要后端工程，是本文档的重点。

### 1.3 MVP 范围（第一版要做的）

- 单门示范课程跑通全流程（建议选机构最熟的一门经济课）
- 学生端：onboarding → 学习计划 → 私教问答（RAG）→ 进度/足迹/薄弱
- 机构端（最简）：上传课件、配置考点与 deadline
- 暂不做：多租户权限体系、家长端、复杂学习分析后台（留待 v2）

---

## 2. 系统架构

### 2.1 总览

```
┌─────────────────────────────────────────────────┐
│                  学生浏览器（前端）                  │
│   聊天 UI · 进度面板 · 足迹 · 薄弱雷达 · 上传        │
└───────────────┬─────────────────────────────────┘
                │ HTTPS
┌───────────────▼─────────────────────────────────┐
│            后端 API（Edge/Serverless Functions）    │
│  • /chat      私教问答（RAG + Claude）              │
│  • /ingest    课件解析入库 + 向量化                  │
│  • /progress  进度/足迹/薄弱读写                     │
│  • /plan      生成学习计划                          │
└───┬──────────────┬──────────────┬────────────────┘
    │              │              │
┌───▼────┐   ┌─────▼─────┐   ┌────▼──────┐
│Supabase│   │Claude API │   │Cloudflare │
│Postgres│   │opus/sonnet│   │   R2      │
│+pgvector│  │           │   │ 课件原文件 │
└────────┘   └───────────┘   └───────────┘
```

### 2.2 技术栈（沿用机构现有积累）

| 层 | 选型 | 说明 |
|----|------|------|
| 前端 | HTML/JS（原型）→ 建议升级 React | 原型已可用，规模化时迁 React |
| 部署 | Netlify | 机构已在用 |
| 后端 | Supabase Edge Functions（Deno）| 与数据库同体系，省运维 |
| 数据库 | Supabase Postgres | 机构已在用 |
| 向量检索 | pgvector（Postgres 扩展）| 无需额外向量库 |
| 文件存储 | Cloudflare R2 | 机构已在用，存课件原件 |
| AI | Claude API | `claude-opus-4-8` 主力问答，`claude-sonnet-4-6` 解析/分类等高频任务 |

**关键判断**：这套栈机构押题系统已经在跑，本产品是"换个用途复用"，不是从零起步——这是最大的工程优势。

---

## 3. 数据库设计（Supabase Postgres）

以下为 MVP 核心表。字段用 snake_case，主键统一 `id uuid default gen_random_uuid()`。

### 3.1 表结构

**students（学生）**
```sql
create table students (
  id uuid primary key default gen_random_uuid(),
  name text,
  email text unique,
  school text,
  base_level text,          -- zero/some/exam/rescue
  daily_minutes int,        -- 每日可投入分钟数
  created_at timestamptz default now()
);
```

**courses（课程）**
```sql
create table courses (
  id uuid primary key default gen_random_uuid(),
  code text,                -- 如 ECON7001
  name text,
  school text,
  exam_format text,         -- exam/essay/viva/mixed
  created_at timestamptz default now()
);
```

**materials（课程资料 + 向量，用于 RAG）**
```sql
create extension if not exists vector;
create table materials (
  id uuid primary key default gen_random_uuid(),
  course_id uuid references courses(id),
  source_file text,         -- R2 中原文件路径
  chunk_index int,          -- 切片序号
  content text,             -- 切片文本
  embedding vector(1024),   -- 向量（维度随 embedding 模型）
  created_at timestamptz default now()
);
create index on materials using ivfflat (embedding vector_cosine_ops);
```

**exam_topics（考点清单，对标押题考点梳理）**
```sql
create table exam_topics (
  id uuid primary key default gen_random_uuid(),
  course_id uuid references courses(id),
  name text,                -- 如"需求与供给弹性"
  keywords text[],          -- 命中关键词，用于话题归类
  weight int default 1      -- 考试权重/重要度
);
```

**student_progress（每生每考点掌握度）**
```sql
create table student_progress (
  id uuid primary key default gen_random_uuid(),
  student_id uuid references students(id),
  topic_id uuid references exam_topics(id),
  level int default 0,      -- 0未开始 1能复述 2能应用 3模拟考通过
  touches int default 0,    -- 练习次数
  updated_at timestamptz default now(),
  unique(student_id, topic_id)
);
```

**chat_logs（学习足迹 — 对话记录存储）**
```sql
create table chat_logs (
  id uuid primary key default gen_random_uuid(),
  student_id uuid references students(id),
  course_id uuid references courses(id),
  topic_id uuid references exam_topics(id),  -- 本轮归属考点，可空
  question text,
  answer text,
  created_at timestamptz default now()
);
```

**weak_spots（薄弱雷达）**
```sql
create table weak_spots (
  id uuid primary key default gen_random_uuid(),
  student_id uuid references students(id),
  topic_id uuid references exam_topics(id),
  count int default 0,            -- 累计卡顿次数
  last_reason text,               -- 最近一次卡在哪
  resolved boolean default false, -- 是否已复习攻克
  updated_at timestamptz default now(),
  unique(student_id, topic_id)
);
```

**deadlines（作业与考试时间）**
```sql
create table deadlines (
  id uuid primary key default gen_random_uuid(),
  course_id uuid references courses(id),
  title text,
  due_date date,
  type text,                -- assign/exam
  weight text               -- 如"15%"
);
```

### 3.2 数据流（学生问一句话发生了什么）

1. 前端把问题发到 `/chat`
2. 后端向量化问题 → 在 `materials` 里检索 top-k 相关切片（RAG）
3. 把切片 + 私教系统 prompt + 该生历史，发给 Claude
4. Claude 返回回答 → 写入 `chat_logs`（足迹）
5. 后端用 Claude 判断本轮归属考点 + 学生是否卡顿：
   - 更新 `student_progress`（掌握度）
   - 若卡顿，更新 `weak_spots`（薄弱雷达）
6. 前端刷新进度/足迹/薄弱三个面板

---

## 4. RAG 检索增强（核心壁垒，重点工程）

RAG 是"本系统答得准、通用 ChatGPT 答得泛"的关键。分两条链路。

### 4.1 入库链路（/ingest，机构上传课件时触发）

```
课件(PPT/PDF/Word) 
  → 上传到 R2 存原件
  → 文本提取（PDF/PPTX 解析）
  → 切片（按语义或固定长度，约 300-500 字/片，带重叠）
  → 调 embedding 模型向量化每一片
  → 写入 materials 表（content + embedding）
  → 同时用 Claude 抽取本课考点 → 写入 exam_topics
```

要点：
- 切片不宜过大（检索精度下降）也不宜过小（上下文割裂），300-500 字是经验起点，需实测调优。
- 考点抽取可用 `claude-sonnet-4-6` 批量做，成本低。

### 4.2 检索链路（/chat 时触发）

```
学生问题 → 向量化 → pgvector 余弦相似度检索 top-5 切片
  → 拼进 prompt 的「参考资料」区
  → Claude 基于资料作答（明确要求"优先依据资料，资料外的标注说明"）
```

参考 SQL（pgvector 检索）：
```sql
select content, 1 - (embedding <=> $1) as score
from materials
where course_id = $2
order by embedding <=> $1
limit 5;
```

### 4.3 注意

- embedding 模型需固定一个（入库和检索必须同一个模型、同一维度）。
- 课件含版权，原件存 R2 私有桶，前端永不直接暴露原文件链接。

---

## 5. 私教教学逻辑（Prompt 工程）

教学"人设"由系统 prompt 控制。原型里已有可用版本（见 `prototypes/latest.html` 的 `sysPrompt()`），正式版在其基础上注入 RAG 资料和学生状态。

系统 prompt 应包含：
1. 角色与课程、学校
2. 学生基础水平（zero/some/exam/rescue）对应的教学深度
3. **检索到的参考资料**（RAG 注入）
4. 教学规则：简洁（200 字内）、举例、讲完追问"检验理解"、对标考试易错点
5. 该生当前薄弱考点（让 AI 主动关照）
6. 输出格式约定

掌握度与薄弱的判定，正式版**改由 Claude 评估**学生回答质量（而非原型的关键词匹配）：每轮对话后追加一次轻量调用，让模型输出结构化 JSON（归属考点、掌握度变化、是否卡顿、卡顿原因），后端据此更新数据库。

---

## 6. 后端 API 设计

| 端点 | 方法 | 作用 | 主要逻辑 |
|------|------|------|---------|
| `/chat` | POST | 私教问答 | RAG 检索 + Claude + 写足迹 + 更新进度/薄弱 |
| `/ingest` | POST | 课件入库 | 解析切片向量化 + 抽考点 |
| `/plan` | POST | 生成学习计划 | 据考点 + deadline + 每日时长，Claude 生成排期 |
| `/progress` | GET | 拉取面板数据 | 返回进度、足迹、薄弱 |
| `/deadline` | POST/GET | 作业考试时间 | 增删查 |

安全：所有端点校验学生身份（Supabase Auth）；Claude API key 只存后端环境变量，**绝不进前端**（原型用的 artifact 内调用仅供演示，正式版必须走后端代理）。

---

## 7. 开发排期建议（务实分阶段）

### 阶段 0 · 验证（1-2 周，几乎不写代码）
- 用 **Dify**（机构已有经验）拿一门课搭原型知识库
- 配教学 prompt，发链接给少量学生试用一周
- 目标：验证学生愿用、效果好，再投开发

### 阶段 1 · MVP 后端（3-4 周）
- 建库（第 3 节所有表）
- 打通 `/ingest`：课件 → R2 → 切片 → pgvector
- 打通 `/chat`：RAG + Claude + 写足迹
- 前端接真后端（替换原型里的 artifact 直连）

### 阶段 2 · 智能闭环（2-3 周）
- 掌握度/薄弱改为 Claude 评估
- `/plan` 动态学习计划
- 进度/足迹/薄弱面板接真数据

### 阶段 3 · 机构端与多课程（2-3 周）
- 机构后台：上传课件、配考点与 deadline
- 批量加课，跑通 2-3 门

### 阶段 4 · 打磨上线（持续）
- RAG 调优（切片、top-k、prompt）
- 命中率/学生反馈回收
- 视情况做家长端、学习分析

---

## 8. 成本与风控提示

- **AI 成本**：每轮对话含 RAG 主问答 + 一次评估调用。高频的评估/分类用 `sonnet` 降本；主问答用 `opus` 保质。可设每生每日额度。
- **版权与隐私**：课件原件私有存储；学生数据按合规要求处理；`.gitignore` 已排除 `materials/`、`.env`。
- **效果可证明**：进度与考点对齐数据是向学生/家长证明效果、写进招生宣传的关键资产，优先保证准确。
- **避免过度承诺**：系统是"显著提升通过概率的自学工具"，不宜对外承诺"保证通过"，以免合规与口碑风险。

---

## 9. 与现有机构系统的协同

- **押题系统**：考点清单、真题、命中率经验可直接喂给本系统的 `exam_topics` 与 RAG 资料。
- **ExamHub 知识库**：可作为 `/ingest` 的资料来源之一。
- **QC/质检与文献检测**：未来可作为"作业辅导"扩展模块接入同一学生档案。

本产品不是孤立的新项目，而是机构既有 AI 资产的一个面向 C 端学生的统一出口。

---

*文档随产品演进更新，改动走 Git 提交记录。*
