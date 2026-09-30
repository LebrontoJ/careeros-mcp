# CareerOS MCP — 开发计划

> 一个本地优先、配置化的 MCP 系统，覆盖求职投递追踪、邮件分类、简历–JD 匹配与改写、申请问题回答、LeetCode 复习、心得记录与日程规划。

- 开始日期：2026-09-30
- 目标：第一阶段（求职核心）在 10 月底前可演示、可写入简历；全部功能在 12 月上旬完成

---

## 1. 项目目标

**功能目标**

- 自动追踪投递状态：从 Gmail 拉取邮件，识别拒信 / OA / 面试邀请 / offer，更新投递记录
- 简历–JD 匹配度评估、关键词缺口分析，以及基于真实经历的 bullet 改写
- 基于个人经历库回答申请表问题（Why us、行为题等），并标注引用来源
- LeetCode 做题记录、间隔重复复习、薄弱题型分析
- 面试复盘与心得记录，自动回流到题库与回答库
- 汇总各类截止日期，生成日程并写入 Google Calendar

**工程目标（简历亮点）**

- 多个自建 MCP server + 统一编排层，支持跨 server 自动化工作流
- 本地优先：敏感数据本地处理，按任务路由本地 / 云端模型，发往云端前脱敏
- 完整的评测体系：每个 server 有独立评测集和量化指标
- 安全设计：邮件内容作为不可信输入处理，有副作用的操作需要人工审批
- 可观测性：tool call 链路追踪、每个工作流的延迟与 token 成本统计

---

## 2. 系统架构

```
          ┌──────────────────────────────────────────────┐
 Client   │   CareerOS Host (CLI / Web)  ·  Claude Desktop  │
          └───────────────────────┬──────────────────────┘
          ┌───────────────────────▼──────────────────────┐
 Orchestr │ Router · Workflow Engine · Approval Queue       │
          │ Tracing · Cost Accounting · Model Router        │
          └──┬───────┬────────┬────────┬───────┬───────┬──┘
 MCP      career   resume   answer   leetcode  notes  planner
 Servers  tracker  server   bank     server    server server
             │        │        │        │        │       │
          SQLite   SQLite   SQLite   SQLite   SQLite  Google Calendar MCP
          (每个 server 独立存储，跨 server 只通过 tool / resource 交互)
```

**设计原则**

- 每个 server 拥有自己的数据，不直接读取其他 server 的表
- 同一套代码支持 stdio 与 streamable HTTP 两种 transport
- 所有 server 可通过配置文件单独开关
- Gmail / Calendar 底层读写复用现成接口，精力集中在领域逻辑与编排

---

## 3. 技术栈

| 类别 | 选型 |
|---|---|
| 语言 / 包管理 | Python 3.12, uv |
| MCP | 官方 Python SDK (FastMCP) |
| 数据校验 | Pydantic v2 |
| 存储 | SQLite + sqlite-vec（向量检索） |
| 本地模型 | Ollama（Qwen2.5 7B 或同级别模型） |
| 云端模型 | Anthropic / OpenAI SDK，统一封装在 Model Router 中 |
| 外部集成 | Gmail API, Google Calendar API (OAuth, token 存本地) |
| 日程求解 | 约束校验器（先自写贪心 + 规则校验，必要时引入 OR-Tools） |
| 可观测性 | OpenTelemetry tracing, Prometheus metrics |
| 测试 / CI | pytest, GitHub Actions |
| 部署 | stdio（默认），Docker Compose（HTTP 模式） |

---

## 4. 仓库结构

```
careeros-mcp/
├── careeros.example.yaml
├── pyproject.toml
├── docker-compose.yml
├── packages/
│   ├── core/                 # 配置加载、Model Router、脱敏、存储基类、tracing
│   ├── host/                 # 编排层：工作流引擎、审批队列、CLI
│   └── servers/
│       ├── career_tracker/
│       ├── resume/
│       ├── answer_bank/
│       ├── leetcode/
│       ├── notes/
│       └── planner/
├── evals/
│   ├── datasets/             # 标注数据（已脱敏，真实数据不入库）
│   ├── injection_suite/      # prompt injection 测试邮件
│   └── run_evals.py
├── docs/
│   ├── architecture.md
│   └── claude_desktop_setup.md
└── tests/
```

---

## 5. 配置文件

```yaml
storage:
  root: ~/.careeros/data
  vector: sqlite-vec

models:
  local:  { provider: ollama, model: qwen2.5:7b }
  cloud:  { provider: anthropic, model: <model-name>, api_key_env: ANTHROPIC_API_KEY }
  routing:
    email_classify: local
    jd_parse: local
    bullet_rewrite: cloud
    answer_generate: cloud
  privacy:
    redact_before_cloud: true

servers:
  career_tracker: { enabled: true,  transport: stdio }
  resume:         { enabled: true,  transport: stdio }
  answer_bank:    { enabled: true,  transport: stdio }
  leetcode:       { enabled: false, transport: stdio }
  notes:          { enabled: false, transport: stdio }
  planner:        { enabled: false, transport: http, port: 8106 }

integrations:
  gmail:    { oauth_token: ~/.careeros/secrets/google.json, poll_minutes: 15 }
  calendar: { oauth_token: ~/.careeros/secrets/google.json }

approval:
  auto: [read]
  require: [calendar_write, email_send, sheet_write]
```

---

## 6. Server 设计

### 6.1 Career Tracker

| 类型 | 名称 | 说明 |
|---|---|---|
| tool | `add_application` | 新增投递（公司、岗位、JD、简历版本、渠道） |
| tool | `update_status` | 更新状态并记录状态流转事件 |
| tool | `search_applications` | 按公司 / 状态 / 时间筛选 |
| tool | `ingest_emails` | 拉取新邮件，分类、抽取、关联投递记录 |
| tool | `review_pending` | 列出低置信度、待人工确认的邮件结果 |
| resource | `tracker://pipeline/summary` | 各状态数量、回复率、平均响应时间 |
| resource | `tracker://applications/{id}` | 单条投递详情与邮件时间线 |

**邮件处理流程**：拉取 → 本地模型分类（rejection / oa / interview / offer / other）→ 结构化抽取（公司、岗位、截止时间）→ 模糊匹配投递记录 → 置信度高自动更新，低则进入待确认队列。

**安全**：分类和抽取阶段的模型不能调用任何 tool，只输出 Pydantic 校验过的结构化结果；邮件原文不会进入拥有 tool 权限的上下文。

**数据表**：`applications`, `status_events`, `emails`

### 6.2 Resume Server

| 类型 | 名称 | 说明 |
|---|---|---|
| tool | `parse_resume` | 解析现有简历为结构化数据 |
| tool | `add_fact` | 向经历事实库添加事实（经历、数字、技术栈） |
| tool | `parse_jd` | 抽取 JD 的技能要求、职责、关键词 |
| tool | `score_match` | 匹配度评分（技能覆盖 + 语义相似度 + LLM 评审） |
| tool | `keyword_gaps` | 列出 JD 中缺失且事实库能支撑的关键词 |
| tool | `rewrite_bullets` | 基于事实库改写 bullet，只允许引用已有事实 |
| tool | `verify_claims` | 校验改写结果中每条陈述是否有事实支撑 |
| tool | `render_resume` | 按现有简历模板渲染，保持原格式与链接不变 |
| resource | `resume://facts` | 经历事实库（只读，供 Answer Bank 使用） |
| resource | `resume://versions/{id}` | 历史简历版本 |

**数据表**：`facts`, `resume_versions`, `jd_analyses`

### 6.3 Answer Bank

| 类型 | 名称 | 说明 |
|---|---|---|
| tool | `answer_question` | 检索经历与历史回答，生成答案并标注引用 |
| tool | `save_final_answer` | 保存人工修改后的最终版本，回流进库 |
| tool | `search_stories` | 按主题检索经历故事（领导力、冲突、失败等） |
| resource | `answers://stories` | 行为面试故事库 |

### 6.4 LeetCode Server

| 类型 | 名称 | 说明 |
|---|---|---|
| tool | `log_problem` | 记录题号、题型标签、用时、卡点、是否独立完成 |
| tool | `due_reviews` | 返回今日到期的复习题（间隔重复算法） |
| tool | `weak_patterns` | 统计薄弱题型 |
| tool | `recommend_problems` | 根据薄弱项与目标公司推荐练习 |

数据以手动记录为主，不依赖非官方接口。

### 6.5 Notes Server

| 类型 | 名称 | 说明 |
|---|---|---|
| tool | `add_note` | 记录心得 |
| tool | `add_debrief` | 面试复盘：技术题回流 LeetCode，行为题回流 Answer Bank |
| tool | `search_notes` | 语义检索历史记录 |

### 6.6 Planner

| 类型 | 名称 | 说明 |
|---|---|---|
| tool | `collect_deadlines` | 汇总 OA 截止、面试时间、复习计划 |
| tool | `propose_schedule` | LLM 生成日程方案 |
| tool | `validate_schedule` | 确定性校验：无冲突、不超过每日可用时长、截止前完成 |
| tool | `commit_schedule` | 写入 Google Calendar（需人工审批） |

---

## 7. 核心跨 server 工作流

**W1：面试邀请**
收到邀请邮件 → Tracker 更新状态 → Resume 读取该岗位 JD → LeetCode 对比薄弱题型 → Planner 在面试前排好练习 → 用户审批后写入日历

**W2：投递前准备**
粘贴 JD → Resume 评分与关键词缺口 → 改写 bullet 并校验 → 渲染定制简历 → Answer Bank 预生成常见申请问题 → Tracker 记录投递及所用简历版本

**W3：面试复盘**
写复盘 → Notes 拆分 → 技术题更新 LeetCode 薄弱项 → 行为题回答回流 Answer Bank

---

## 8. 评测计划

| 模块 | 评测集 | 指标 |
|---|---|---|
| 邮件分类 | 约 100 封真实求职邮件（本地标注，不上传） | accuracy, macro-F1 |
| 字段抽取 | 同上 | 字段级精确匹配率 |
| 投递关联 | 同上 | 关联准确率、待确认比例 |
| Prompt injection | 20+ 构造的恶意邮件 | 攻击成功率（目标为 0） |
| Bullet 改写 | 30 组 JD × 原 bullet | 无事实支撑陈述的拦截率、关键词覆盖提升 |
| 匹配度评分 | 自己的真实投递结果 | 评分与是否获得回复的相关性（样本量足够后） |
| 申请回答 | 20 个常见问题 | 引用正确率、人工评分 |
| 模型路由 | 以上各任务 | 本地 vs 云端的准确率、延迟、单次成本对比 |

评测脚本接入 CI，每次修改 prompt 或模型配置都自动回归。

---

## 9. 里程碑

### Phase 0：项目骨架（9/30 – 10/6）

- [ ] 初始化仓库、uv 工作区、pre-commit、GitHub Actions
- [ ] `core`：配置加载与校验、Model Router、脱敏模块、SQLite 基类
- [ ] 最小可用的 Career Tracker server（`add_application` / `search_applications`）
- [ ] 接入 Claude Desktop 跑通 stdio 模式
- [ ] `careeros up` 命令按配置启动 server

### Phase 1：求职核心（10/7 – 10/27）★ 简历里程碑

- [ ] Gmail OAuth 与增量拉取
- [ ] 邮件分类、抽取、关联、待确认队列
- [ ] 隔离式邮件处理与 injection 测试集
- [ ] Resume Server：事实库、JD 解析、匹配评分、关键词缺口
- [ ] Bullet 改写 + `verify_claims` 校验
- [ ] 按现有模板渲染简历
- [ ] 邮件与改写两组评测集及首轮结果
- [ ] README 初版 + 演示 GIF
- [ ] 写入简历并开始投递

### Phase 2：知识回流（10/28 – 11/17）

- [ ] Answer Bank：RAG 检索、引用标注、最终版本回流
- [ ] LeetCode Server：记录、间隔重复、薄弱题型
- [ ] Notes Server：复盘拆分与回流
- [ ] 工作流 W2、W3
- [ ] 申请回答评测

### Phase 3：编排与部署（11/18 – 12/8）

- [ ] Planner：截止日期汇总、方案生成、确定性校验、审批写入
- [ ] Host 工作流引擎与审批队列
- [ ] 工作流 W1
- [ ] HTTP transport + Docker Compose
- [ ] OpenTelemetry 链路追踪、Prometheus 指标、成本统计
- [ ] 本地 vs 云端模型完整对比报告

### Phase 4：打磨（持续）

- [ ] 架构文档与设计取舍说明
- [ ] 演示视频
- [ ] 技术博客（可选）

每个阶段预留约 20% 时间作为缓冲。

---

## 10. 简历 bullet 方向（待填入真实数据）

- 架构：设计本地优先的多 server MCP 系统，N 个领域 server 通过统一编排层完成跨服务工作流
- 安全与可靠性：隔离式处理不可信邮件输入，injection 测试集攻击成功率 X%；有副作用操作经人工审批
- 评测与模型路由：构建 N 组评测集，按任务路由本地 / 云端模型，在准确率下降不超过 X% 的前提下降低 Y% 成本
- 防幻觉：基于经历事实库的简历改写与校验，拦截 X% 无依据陈述

---

## 11. 风险与应对

| 风险 | 应对 |
|---|---|
| 范围过大，影响求职节奏 | 严格按阶段推进，Phase 1 完成即可写入简历 |
| 本地模型效果不足 | 按任务路由，写作类任务交给云端模型 |
| 真实邮件与简历数据泄露 | 数据与 token 仅存本地并加入 `.gitignore`；公开仓库只含脱敏样例 |
| 评测样本量不足 | 先用小规模标注集，随使用持续补充 |
| Google API 配额与 OAuth 配置复杂 | 增量拉取 + 本地缓存，文档中记录配置步骤 |
