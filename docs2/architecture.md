# CrossBeam 系统架构设计文档

> **项目简介**：CrossBeam 是一个 AI 驱动的加利福尼亚州 ADU（附属住宅单元）许可证助手，利用 Claude Opus 4.6 解决首次提交拒绝率高达 90% 的许可审批瓶颈，帮助承包商和房主节省 $30,000+ 的延误成本。
>
> 本系统荣获 Anthropic"Built with Opus 4.6" Claude Code Hackathon（2026 年 2 月）冠军。

---

## 目录

1. [系统概述](#1-系统概述)
2. [技术栈](#2-技术栈)
3. [整体架构图](#3-整体架构图)
4. [目录结构](#4-目录结构)
5. [核心模块详解](#5-核心模块详解)
6. [数据模型](#6-数据模型)
7. [API 设计](#7-api-设计)
8. [认证与授权](#8-认证与授权)
9. [AI Agent 架构](#9-ai-agent-架构)
10. [数据流详解](#10-数据流详解)
11. [基础设施与部署](#11-基础设施与部署)
12. [外部服务集成](#12-外部服务集成)
13. [关键架构决策](#13-关键架构决策)

---

## 1. 系统概述

### 业务背景

加州 ADU 许可证申请首次通过率极低，主要原因是：
- 规划图纸不符合加州 HCD 手册或当地市政法规
- 承包商对复杂的更正信（Corrections Letter）难以有效回应
- 每次往返审批周期长达 2-3 个月，造成 $30,000+ 延误成本

### 两条核心业务流

| 流程 | 用户角色 | 描述 |
|------|---------|------|
| **更正分析流（Corrections Analysis）** | 承包商 | 上传更正信 + 规划图 → Agent 解析法规 → 生成问题清单 → 生成专业回复包 |
| **城市审查流（City Review）** | 城市建筑部门 | 上传申请文件 → Agent 对照州法 + 市政法规审查 → 生成更正信草稿 |

---

## 2. 技术栈

| 层级 | 技术 | 用途 |
|------|------|------|
| **前端** | Next.js 16 + React 19 | SPA + SSR + API Routes |
| **UI 组件** | shadcn/ui + Tailwind CSS 4 | 设计系统 |
| **后端编排** | Express 5 + TypeScript | Cloud Run 编排服务 |
| **AI 模型** | Claude Opus 4.6 | 核心 Agent 推理 |
| **Agent 运行时** | Anthropic Agent SDK | 多 Agent 编排 |
| **沙箱环境** | Vercel Sandbox | Agent 隔离执行环境 |
| **数据库** | Supabase (PostgreSQL) | 数据持久化 |
| **实时推送** | Supabase Realtime | WebSocket 状态更新 |
| **文件存储** | Supabase Storage | PDF、PNG、输出文件 |
| **认证** | Supabase Auth (JWT) | 用户认证 |
| **PDF 处理** | pdftoppm + ImageMagick | PDF → PNG 转换 |
| **部署** | Vercel (前端) + GCP Cloud Run (后端) | 生产环境 |

---

## 3. 整体架构图

```
┌──────────────────────────────────────────────────────────────────────┐
│                         用户浏览器                                    │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    Next.js 前端 (Vercel)                     │    │
│  │                                                             │    │
│  │  ┌─────────────┐  ┌──────────────┐  ┌───────────────────┐  │    │
│  │  │  项目管理页  │  │  Agent 流式  │  │  承包商问答表单   │  │    │
│  │  │  my-projects│  │  消息流展示  │  │  (Q&A Form)       │  │    │
│  │  └─────────────┘  └──────────────┘  └───────────────────┘  │    │
│  │                                                             │    │
│  │  ┌─────────────────────────────────────────────────────┐   │    │
│  │  │              Next.js API Routes                      │   │    │
│  │  │  /api/generate  /api/projects/[id]  /api/extract    │   │    │
│  │  └────────────────────────┬────────────────────────────┘   │    │
│  └───────────────────────────┼─────────────────────────────────┘    │
└──────────────────────────────┼───────────────────────────────────────┘
                               │ HTTP
                               ▼
┌──────────────────────────────────────────────────────────────────────┐
│                    Express 编排服务 (GCP Cloud Run)                   │
│                                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────────────┐    │
│  │ /generate    │  │ /extract     │  │     Sandbox 管理         │    │
│  │ 触发 Agent   │  │ PDF → PNG    │  │  (创建/等待/清理)        │    │
│  │ 流           │  │ 转换服务     │  │                          │    │
│  └──────┬───────┘  └──────────────┘  └────────────────────────┘    │
└─────────┼────────────────────────────────────────────────────────────┘
          │ Vercel Sandbox SDK
          ▼
┌──────────────────────────────────────────────────────────────────────┐
│                    Vercel Sandbox (隔离执行环境)                       │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                   Agent SDK + Claude Opus 4.6                │   │
│  │                                                              │   │
│  │  ┌─────────────────────────────────────────────────────┐    │   │
│  │  │  主 Agent (Lead Agent)                               │    │   │
│  │  │  ├── 读取更正信（视觉）                              │    │   │
│  │  │  ├── 解析每条更正项                                  │    │   │
│  │  │  ├── 启动并发子 Agent（滚动窗口 × 3）               │    │   │
│  │  │  │     ├─ 状态法规研究子 Agent                      │    │   │
│  │  │  │     ├─ 城市法规研究子 Agent                      │    │   │
│  │  │  │     └─ 规划图页面分析子 Agent                    │    │   │
│  │  │  └── 生成结构化输出                                  │    │   │
│  │  └─────────────────────────────────────────────────────┘    │   │
│  │                                                              │   │
│  │  技能库 (Skills): california-adu / adu-corrections-flow /    │   │
│  │                   adu-city-research / adu-plan-review / ...  │   │
│  └──────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────┘
          │                               │
          │ 读写                           │ 实时推送
          ▼                               ▼
┌───────────────────────────────────────────────────────────────────┐
│                         Supabase                                   │
│                                                                   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐   │
│  │  PostgreSQL  │  │   Storage    │  │  Realtime (WebSocket)│   │
│  │  - projects  │  │  - uploads   │  │  - projects 变更     │   │
│  │  - files     │  │  - outputs   │  │  - messages 新增     │   │
│  │  - messages  │  │  - archives  │  │  - outputs 创建      │   │
│  │  - outputs   │  │              │  │                      │   │
│  │  - answers   │  │              │  │                      │   │
│  └──────────────┘  └──────────────┘  └──────────────────────┘   │
└───────────────────────────────────────────────────────────────────┘
```

---

## 4. 目录结构

```
crossbeam/
├── frontend/                          # Next.js 应用（Vercel 部署）
│   ├── app/
│   │   ├── (auth)/login/             # Supabase 登录页
│   │   ├── (dashboard)/              # 登录后主应用
│   │   │   ├── dashboard/            # 角色选择（城市/承包商）
│   │   │   ├── my-projects/          # 项目列表
│   │   │   └── projects/[id]/        # 项目详情 + 结果展示
│   │   ├── api/                      # Next.js API 路由
│   │   │   ├── generate/             # 触发 Agent（调用 Cloud Run）
│   │   │   ├── projects/[id]/        # 项目数据 + Realtime
│   │   │   └── extract/              # PDF 提取代理
│   │   └── page.tsx                  # 落地页
│   ├── components/                   # React 组件
│   │   ├── persona-card.tsx          # 角色选择卡
│   │   ├── agent-stream.tsx          # 实时消息流
│   │   └── contractor-questions-form.tsx  # 承包商问答表单
│   ├── lib/
│   │   ├── supabase/                 # Supabase 客户端 + 认证
│   │   └── api-auth.ts               # 双模认证（Session + API Key）
│   └── types/
│       └── database.ts               # Supabase 类型定义
│
├── server/                            # Express 编排服务（Cloud Run）
│   ├── src/
│   │   ├── index.ts                  # Express 入口
│   │   ├── routes/
│   │   │   ├── generate.ts           # POST /generate — 触发 Agent 流
│   │   │   └── extract.ts            # POST /extract — PDF 提取
│   │   ├── services/
│   │   │   ├── sandbox.ts            # Vercel Sandbox 生命周期管理
│   │   │   ├── supabase.ts           # Supabase 客户端 + DB 操作
│   │   │   └── extract.ts            # PDF → PNG 转换
│   │   └── utils/
│   │       └── config.ts             # 流程配置、提示词、预算
│   └── skills/                       # Agent 技能库（知识库）
│       ├── california-adu/           # 28 个加州 ADU 法规参考文件
│       ├── adu-corrections-flow/     # 更正分析编排技能
│       ├── adu-corrections-complete/ # 第二阶段：回复生成技能
│       ├── adu-city-research/        # 城市法规研究技能
│       ├── adu-plan-review/          # 城市侧审查技能
│       ├── adu-targeted-page-viewer/ # 视觉分析页面技能
│       ├── placentia-adu/            # 城市专项：Placentia
│       └── buena-park-adu/           # 城市专项：Buena Park
│
├── agents-crossbeam/                 # Agent SDK 测试/示例流
│   ├── src/flows/                    # 示例流程
│   └── src/tests/                    # 集成测试
│
├── docs/                             # 研究文档、架构笔记
├── docs2/                            # 架构设计文档（本文档所在位置）
├── test-assets/                      # 真实许可证数据（测试用）
│   ├── corrections/                  # Placentia 更正信 + 规划图
│   └── approved/                     # Long Beach 已批准规划
│
└── .claude/skills/                   # Claude Code 本地技能
    ├── demo-city-review/             # 演示：城市侧审查
    └── demo-contractor-corrections/  # 演示：承包商更正流程
```

---

## 5. 核心模块详解

### 5.1 前端模块（frontend/）

#### 路由结构

| 路由 | 描述 |
|------|------|
| `/` | 落地页（产品介绍）|
| `/login` | Supabase 邮箱/密码登录 |
| `/dashboard` | 角色选择（城市建筑部门 / 承包商）|
| `/my-projects` | 项目列表（卡片视图）|
| `/projects/[id]` | 项目详情：文件上传、Agent 消息流、结果展示 |

#### 关键组件

| 组件 | 功能 |
|------|------|
| `persona-card.tsx` | 城市/承包商角色选择卡 |
| `agent-stream.tsx` | 实时 Agent 消息流展示（Realtime 订阅）|
| `contractor-questions-form.tsx` | 承包商问题问答交互表单 |
| `file-upload.tsx` | PDF + 图片上传（Supabase Storage）|

### 5.2 后端编排模块（server/）

#### 主要服务

| 服务 | 文件 | 职责 |
|------|------|------|
| Sandbox 管理 | `services/sandbox.ts` | 创建、配置、监控、清理 Vercel Sandbox |
| Supabase 操作 | `services/supabase.ts` | 项目状态更新、文件管理、输出写入 |
| PDF 提取 | `services/extract.ts` | pdftoppm + ImageMagick 转换 PDF → PNG |
| 流程配置 | `utils/config.ts` | 预算限制、流程提示词、Agent 参数 |

#### 流程预算配置

```typescript
FLOW_BUDGET = {
  'city-review':            { maxTurns: 500, maxBudgetUsd: 50 },
  'corrections-analysis':   { maxTurns: 500, maxBudgetUsd: 50 },
  'corrections-response':   { maxTurns: 150, maxBudgetUsd: 20 },
}
```

### 5.3 技能库（Skills Architecture）

技能库是系统的核心知识层，将领域知识结构化编码，供 Agent 按需加载。

#### 技能目录

| 技能名 | 类型 | 描述 |
|--------|------|------|
| `california-adu` | 法规参考 | 28 个文件编码加州 HCD 手册 + 政府法典 §§66310-66342 |
| `adu-corrections-flow` | 编排技能 | 更正分析四阶段流程编排 |
| `adu-corrections-complete` | 生成技能 | 第二阶段：回复信、专业范围、注释生成 |
| `adu-city-research` | 研究技能 | 三模式城市法规研究（WebSearch/WebFetch/Browser）|
| `adu-plan-review` | 审查技能 | 城市侧五专业子 Agent 审查编排 |
| `adu-targeted-page-viewer` | 视觉技能 | 规划图页面视觉分析子 Agent |
| `placentia-adu` | 城市专项 | Placentia 市离线法规数据 |
| `buena-park-adu` | 城市专项 | Buena Park 市离线法规数据 |

#### california-adu 决策树路由

```
用户查询
    │
    ▼
地块类型判断（单家庭 / 多家庭 / 商业）
    │
    ▼
建设类型判断（新建 / 改建 / 车库改建 / JADU）
    │
    ▼
适用修饰符（历史区 / 火灾区 / 洪水区）
    │
    ▼
流程类型（行政审批 / 标准审批 / 例外处理）
    │
    ▼
加载 3-5 个相关参考文件（非全部 28 个）
```

---

## 6. 数据模型

### 6.1 核心数据库表（Supabase PostgreSQL）

#### projects 表

```sql
CREATE TABLE crossbeam.projects (
  id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id        UUID NOT NULL REFERENCES auth.users(id),
  flow_type      TEXT CHECK (flow_type IN ('city-review', 'corrections-analysis')),
  project_name   TEXT NOT NULL,
  project_address TEXT,
  city           TEXT,
  status         TEXT CHECK (status IN (
                   'ready', 'uploading', 'processing',
                   'processing-phase1', 'awaiting-answers',
                   'processing-phase2', 'completed', 'failed'
                 )),
  error_message  TEXT,
  is_demo        BOOLEAN DEFAULT false,
  created_at     TIMESTAMPTZ DEFAULT now(),
  updated_at     TIMESTAMPTZ DEFAULT now()
);
```

#### files 表

```sql
CREATE TABLE crossbeam.files (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id   UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  file_type    TEXT CHECK (file_type IN ('plan-binder', 'corrections-letter', 'other')),
  filename     TEXT NOT NULL,
  storage_path TEXT NOT NULL,   -- 'crossbeam-uploads/USER_ID/PROJECT_ID/...'
  mime_type    TEXT,
  size_bytes   BIGINT,
  created_at   TIMESTAMPTZ DEFAULT now()
);
```

#### messages 表（实时 Agent 消息流）

```sql
CREATE TABLE crossbeam.messages (
  id         BIGSERIAL PRIMARY KEY,
  project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  role       TEXT CHECK (role IN ('system', 'assistant', 'tool')),
  content    TEXT NOT NULL,
  created_at TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX ON messages (project_id, created_at);
```

#### outputs 表（Agent 输出，带版本）

```sql
CREATE TABLE crossbeam.outputs (
  id                        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id                UUID NOT NULL REFERENCES projects(id),
  flow_phase                TEXT CHECK (flow_phase IN ('analysis', 'response', 'review')),
  version                   INTEGER,   -- 按 project_id + flow_phase 自动递增

  -- 第一阶段（分析）
  corrections_analysis_json JSONB,
  contractor_questions_json JSONB,

  -- 第二阶段（回复）
  response_letter_md        TEXT,
  professional_scope_md     TEXT,
  corrections_report_md     TEXT,
  sheet_annotations_json    JSONB,

  -- 城市审查
  corrections_letter_md     TEXT,
  review_checklist_json     JSONB,

  -- 元数据
  raw_artifacts             JSONB,
  agent_cost_usd            NUMERIC,
  agent_turns               INTEGER,
  agent_duration_ms         BIGINT,
  created_at                TIMESTAMPTZ DEFAULT now()
);
```

#### contractor_answers 表

```sql
CREATE TABLE crossbeam.contractor_answers (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id    UUID NOT NULL REFERENCES projects(id),
  output_id     UUID REFERENCES outputs(id),
  question_key  TEXT NOT NULL,   -- 'q_4_0'（第4条更正，第0个问题）
  question_text TEXT NOT NULL,
  question_type TEXT CHECK (question_type IN ('text', 'number', 'choice', 'measurement')),
  options       JSONB,           -- choice 类型的选项列表
  answer_text   TEXT,
  is_answered   BOOLEAN DEFAULT false,
  created_at    TIMESTAMPTZ DEFAULT now()
);
```

### 6.2 Supabase Storage Buckets

| Bucket | 用途 |
|--------|------|
| `crossbeam-uploads` | 用户上传的 PDF + 提取的 PNG 压缩包 + Agent 输出 |
| `crossbeam-demo-assets` | Demo 项目预提取数据（Placentia 示例）|
| `crossbeam-outputs` | Agent 沙箱执行期间生成的文件 |

### 6.3 关键 JSON Schema

#### contractor_questions.json（第一阶段输出）

```json
{
  "summary": {
    "total_items": 14,
    "auto_fixable": 5,
    "needs_input": 6,
    "needs_professional": 3
  },
  "question_groups": [
    {
      "correction_item_id": "4",
      "item_summary": "公用设施连接",
      "category": "NEEDS_CONTRACTOR_INPUT",
      "questions": [
        {
          "question_id": "q_4_0",
          "question_text": "现有污水/排水管道的尺寸是多少？",
          "question_type": "choice",
          "options": ["3\" ABS", "4\" ABS", "4\" PVC", "6\" PVC"],
          "context": "CPC Table 702.1 规定：3\" 管允许 18 DFU，4\" 管允许 48 DFU"
        }
      ]
    }
  ],
  "auto_fixable_items": [...],
  "professional_items": [...]
}
```

#### sheet_annotations.json（第二阶段输出）

```json
{
  "annotations": [
    {
      "sheet_id": "A3",
      "page_number": 7,
      "actions": [
        {
          "item_number": "5",
          "area": "南立面，门廊顶棚",
          "action": "更换为 1 小时防火等级顶棚组件",
          "specification": "5/8\" X 型石膏板，符合 CRC R302.1",
          "status": "SCOPED"
        }
      ]
    }
  ]
}
```

---

## 7. API 设计

### 7.1 前端 API Routes（Next.js）

| 端点 | 方法 | 认证 | 描述 |
|------|------|------|------|
| `/api/generate` | POST | Session / API Key | 触发 Agent 流（调用 Cloud Run）|
| `/api/projects/[id]` | GET | Session / API Key | 获取项目详情 + 文件 + 消息 + 最新输出 |
| `/api/extract` | POST | Session / API Key | 代理到 Cloud Run PDF 提取服务 |
| `/api/reset-project` | POST | Session | 重置项目（重跑）|

**POST /api/generate 请求体：**
```json
{
  "project_id": "uuid",
  "user_id": "uuid",
  "flow_type": "corrections-analysis | corrections-response | city-review"
}
```

### 7.2 后端 API（Express Cloud Run）

| 端点 | 方法 | 描述 |
|------|------|------|
| `/generate` | POST | 主编排：创建 Sandbox、下载文件、提取 PDF、运行 Agent |
| `/extract` | POST | PDF 提取服务（pdftoppm + ImageMagick）|
| `/health` | GET | 健康检查 |

**POST /generate 处理流程：**
```
1. 从 Supabase 拉取项目文件（PDF + 更正信）
2. 调用 /extract 将 PDF 转换为 PNG
3. 将文件压缩包上传到 Supabase Storage
4. 创建 Vercel Sandbox
5. 在 Sandbox 中下载文件 + 安装技能
6. 以 detached 模式启动 Agent 命令
7. 轮询等待 Agent 完成（最多 120 次，每 30s 一次）
8. Agent 结果已写入 Supabase → 前端通过 Realtime 收到通知
```

---

## 8. 认证与授权

### 8.1 双模认证设计（api-auth.ts）

```
HTTP Request
     │
     ▼
authenticateRequest()
     │
     ├─ 有 Authorization: Bearer <key> ?
     │     ├─ 是 → API Key 认证（Agent / 服务间调用）
     │     │        跳过所有权检查（信任内部服务）
     │     └─ 否 → Session Cookie 认证（浏览器用户）
     │              JWT 验证 → 返回 user_id
     │
     └─ 所有权检查
           ├─ isApiKey = true → 跳过
           └─ isApiKey = false → project.user_id === userId
```

### 8.2 Supabase RLS（行级安全）

| 表 | 策略 |
|----|------|
| `projects` | 用户只能读写自己的项目（`user_id = auth.uid()`）|
| `files` | 通过 FK 继承 projects RLS |
| `outputs` | 通过 FK 继承 projects RLS |
| `contractor_answers` | 通过 FK 继承 projects RLS |
| `messages` | 通过 FK 继承 projects RLS |

### 8.3 环境变量

```bash
# 前端（Vercel）
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=      # 仅服务端 API Routes
CLOUD_RUN_URL=                  # 后端编排服务地址

# 后端（Cloud Run）
ANTHROPIC_API_KEY=
SUPABASE_URL=
SUPABASE_SERVICE_ROLE_KEY=
VERCEL_TOKEN=
VERCEL_TEAM_ID=
VERCEL_PROJECT_ID=
```

---

## 9. AI Agent 架构

### 9.1 Agent 执行模型

```
Cloud Run 编排服务
    │
    │  创建 Sandbox
    ▼
Vercel Sandbox（隔离文件系统 + 网络）
    │
    │  claude_code preset（启用 Bash / 文件 I/O / 网络工具）
    ▼
主 Agent（Lead Agent）
    │
    ├─ 顺序任务：读取更正信、构建清单
    │
    └─ 并发子 Agent（滚动窗口 × 3）
          ├─ 状态法规研究子 Agent
          │    └─ 加载 california-adu 技能（决策树选 3-5 文件）
          ├─ 城市法规研究子 Agent
          │    └─ 加载 adu-city-research 技能（WebSearch + WebFetch）
          └─ 规划图页面分析子 Agent
               └─ 加载 adu-targeted-page-viewer 技能（视觉分析）
```

### 9.2 子 Agent 滚动窗口策略

**问题**：每个规划图页面是 7,400px × 4,000px+ 的高分辨率 PNG，若所有图片堆积在主 Agent 上下文中，会超出 token 限制。

**方案**：每个页面启动独立的子 Agent（一次只有 3 个并发），子 Agent 完成后上下文释放，图片不累积。

```
更正项 1-14
    │
    ├─ 批次 1（并发）：更正项 1、2、3 各自一个子 Agent
    ├─ 批次 2（并发）：更正项 4、5、6 各自一个子 Agent
    └─ ...
    
每个子 Agent：视觉分析 → 写结果文件 → 退出（上下文清空）
```

### 9.3 城市法规研究三模式（adu-city-research）

| 模式 | 工具 | 时长 | 触发条件 |
|------|------|------|---------|
| Mode 1：发现 | WebSearch | ~30s | 首选，搜索城市 ADU 页面 |
| Mode 2：提取 | WebFetch | 60-90s | 发现 URL 后拉取全文 |
| Mode 3：浏览器回退 | Chrome MCP | 2-3min | JS 渲染页面（如 ecode360）|

### 9.4 Agent 执行参数

```typescript
{
  model: 'claude-opus-4-6',
  preset: 'claude_code',
  maxTurns: 500,
  maxBudgetUsd: 50,
  
  // Detached 模式（防止 GCP 负载均衡器 5 分钟空闲断开）
  detached: true,
  
  // 轮询等待配置
  maxPollingAttempts: 120,    // 最多等待 60 分钟
  pollingIntervalMs: 30_000,  // 每 30 秒检查一次
}
```

---

## 10. 数据流详解

### 10.1 承包商更正分析完整流程

```
用户（承包商）
    │
    │  1. 上传 PDF 规划图 + 更正信 PNG
    ▼
Next.js 前端
    │  POST /api/generate { project_id, flow_type: 'corrections-analysis' }
    ▼
Cloud Run 编排
    │
    ├─ Step 1: 从 Supabase Storage 下载文件
    ├─ Step 2: pdftoppm + ImageMagick 将 PDF 转换为 PNG（15-26 张）
    ├─ Step 3: 压缩打包，上传到 Supabase Storage
    ├─ Step 4: 创建 Vercel Sandbox
    └─ Step 5: 在 Sandbox 中以 detached 模式启动 Agent
    
Vercel Sandbox（Agent 执行，约 10-15 分钟）
    │
    ├─ 读取更正信（视觉模型解析 PNG）
    ├─ 解析 14 条更正项
    ├─ 读取规划图封面，构建图纸索引（Sheet → Page 映射）
    ├─ 滚动窗口并发子 Agent（每批 3 个）：
    │   ├─ 每条更正项 → 查询州法规（california-adu 技能）
    │   ├─ 每条更正项 → 查询城市法规（adu-city-research 技能）
    │   └─ 每条更正项 → 视觉分析相关图纸页面
    ├─ 分类更正项：
    │   ├─ AUTO_FIXABLE（承包商可自行解决）
    │   ├─ NEEDS_CONTRACTOR_INPUT（需要承包商提供信息）
    │   └─ NEEDS_PROFESSIONAL（需要工程师或专业人员）
    ├─ 生成 contractor_questions.json
    └─ 写入 Supabase：
        ├─ outputs 记录（status: 'awaiting-answers'）
        └─ messages 流（Realtime 推送前端）

前端（实时更新）
    │  Supabase Realtime 推送 → UI 刷新
    ▼
    │  展示承包商问答表单
    ▼
承包商填写答案 → contractor_answers 表写入

用户点击"生成回复"
    │  POST /api/generate { flow_type: 'corrections-response' }
    ▼
Cloud Run + Sandbox（第二阶段，约 5-10 分钟）
    │
    └─ 读取第一阶段分析 + 承包商答案
       生成：
       ├─ response_letter.md（专业回复信）
       ├─ professional_scope.md（各专业需做的工作）
       ├─ corrections_report.md（更正状态仪表板）
       └─ sheet_annotations.json（每张图纸的修改注释）
       
       → 写入 Supabase outputs → Realtime 推送 → 前端展示结果
```

### 10.2 项目状态机

```
ready
  │
  ▼（上传文件）
uploading
  │
  ▼（触发 Agent）
processing-phase1
  │
  ▼（Agent 完成第一阶段）
awaiting-answers
  │
  ▼（用户提交答案 + 触发第二阶段）
processing-phase2
  │
  ├─ 成功 → completed
  └─ 失败 → failed
```

---

## 11. 基础设施与部署

### 11.1 三层基础设施

```
┌─────────────────────────────────────────────────────────────┐
│  层 1：Vercel（前端）                                        │
│  ├─ Next.js 16 静态 + SSR                                  │
│  ├─ API Routes（/api/*）                                    │
│  └─ 自动 HTTPS + CDN                                       │
└─────────────────────────────────────────────────────────────┘
           │ HTTP 请求（长连接不需要，有 Realtime）
┌─────────────────────────────────────────────────────────────┐
│  层 2：GCP Cloud Run（后端编排）                             │
│  ├─ Express 5 + TypeScript                                 │
│  ├─ 系统依赖：pdftoppm, ImageMagick                        │
│  ├─ 持久化进程（不受 Vercel 函数超时限制）                   │
│  └─ Detached Sandbox 模式（抗 GCP 负载均衡 5min 断连）       │
└─────────────────────────────────────────────────────────────┘
           │ Vercel Sandbox SDK
┌─────────────────────────────────────────────────────────────┐
│  层 3：Vercel Sandbox（Agent 执行）                          │
│  ├─ 隔离 Linux 环境（每次任务独立）                          │
│  ├─ 文件系统访问（claude_code preset）                      │
│  ├─ 30 分钟超时上限                                         │
│  └─ 自动清理（任务完成后）                                   │
└─────────────────────────────────────────────────────────────┘
```

### 11.2 为何选择该基础设施组合

| 问题 | 解决方案 |
|------|---------|
| Agent 运行需 10-30 分钟，超出 Vercel 函数 60-300s 超时 | Cloud Run 作为持久化编排层 |
| Agent 需要文件系统访问（PDF → PNG → Agent 读取）| Vercel Sandbox（隔离文件系统）|
| GCP 负载均衡器 5 分钟空闲连接断开 | Sandbox detached 模式 + 轮询等待 |
| 前端需要实时状态更新（无轮询开销）| Supabase Realtime WebSocket |
| 多个请求并发时需要隔离 | 每个任务独立 Sandbox 实例 |

### 11.3 典型执行指标

| 指标 | 数值 |
|------|------|
| 总执行时长 | 10-15 分钟（优化后，之前 35 分钟）|
| Agent 费用 | ~$3-5/次（Opus 4.6 定价）|
| Agent 轮次 | 50-100 turns |
| PDF 尺寸 | 15-26 页，每页 7,400px × 4,000px+ |
| 并发子 Agent | 3 个（滚动窗口）|

---

## 12. 外部服务集成

### 12.1 服务清单

| 服务 | 用途 | 使用方 |
|------|------|--------|
| Claude API（Opus 4.6）| Agent 核心推理 | Vercel Sandbox |
| Vercel Sandbox SDK | 隔离执行环境 | Cloud Run |
| Supabase Database | 数据持久化 | 前端 + Cloud Run |
| Supabase Storage | 文件存储 | 前端 + Cloud Run + Agent |
| Supabase Auth | 用户认证 | 前端 |
| Supabase Realtime | 实时推送 | 前端 |
| WebSearch | 城市法规发现 | Agent（adu-city-research）|
| WebFetch | 城市法规提取 | Agent（adu-city-research）|
| Chrome MCP | JavaScript 页面备用 | Agent（adu-city-research）|
| pdftoppm | PDF 渲染 | Cloud Run |
| ImageMagick | 图片处理 | Cloud Run |

### 12.2 Supabase Realtime 订阅设计

```typescript
// 前端订阅三个表的变更
supabase
  .channel(`project-${projectId}`)
  .on('postgres_changes', 
    { event: 'UPDATE', schema: 'crossbeam', table: 'projects', filter: `id=eq.${projectId}` },
    (payload) => updateProjectStatus(payload.new)
  )
  .on('postgres_changes',
    { event: 'INSERT', schema: 'crossbeam', table: 'messages', filter: `project_id=eq.${projectId}` },
    (payload) => appendMessage(payload.new)
  )
  .on('postgres_changes',
    { event: 'INSERT', schema: 'crossbeam', table: 'outputs', filter: `project_id=eq.${projectId}` },
    (payload) => loadOutput(payload.new)
  )
  .subscribe()
```

---

## 13. 关键架构决策

### 决策矩阵

| 决策 | 方案 | 原因 |
|------|------|------|
| Agent 运行时隔离 | Vercel Sandbox | Agent 需要文件系统 + 网络，不能在 Vercel 函数中运行 |
| 长任务编排 | GCP Cloud Run | Vercel/Lambda 超时，需持久化进程 |
| 知识编码 | 结构化技能文件（非 RAG）| 硬编码法规易出错；结构化文件可维护且精确 |
| 决策树路由 | 按需加载 3-5 文件 | 避免一次加载 28 个文件造成上下文膨胀 |
| 子 Agent 图像分析 | 每页独立子 Agent | 高分辨率图像堆积会超出 token 限制 |
| 实时状态更新 | Supabase Realtime | 轮询浪费带宽；Realtime 仅在变更时推送 |
| 双模认证 | Session + API Key | 浏览器用户用 JWT，服务间调用用 API Key |
| Detached Sandbox | 异步轮询模式 | GCP 负载均衡器强制断开空闲连接 |
| PDF 预提取 | Cloud Run 提前处理 | 沙箱专注 AI 推理，不消耗 token 在文件系统工具上 |

### 性能优化历程

| 优化 | 效果 |
|------|------|
| 先读更正信 → 按需查看图纸页面 | 执行时间从 35 分钟降至 10-15 分钟 |
| 子 Agent 滚动窗口（×3）| 避免图像累积 + 加速并发处理 |
| 决策树路由 | 减少 80% 无关法规文件加载 |
| Cloud Run 预提取 PDF | 沙箱启动更快，Agent 专注推理 |
| Detached 模式 | 消除 GCP 连接断开导致任务失败 |

---

## 附录：项目开发状态

| 功能 | 状态 |
|------|------|
| 承包商更正分析（第一阶段）| ✅ 生产可用 |
| 承包商更正回复（第二阶段）| ✅ 生产可用 |
| PDF 提取 + PNG 生成 | ✅ 生产可用 |
| 多 Agent 并发编排 | ✅ 生产可用 |
| Realtime 前端更新 | ✅ 生产可用 |
| 城市侧审查流（UI）| 🔄 开发中 |
| PDF 图纸重绘 | 📋 路线图 |
| 480+ 城市支持 | 📋 路线图 |
| 每用户 API Key | 📋 路线图 |
| 移动端体验 | 📋 路线图 |

---

*文档生成时间：2026-05-11*  
*文档版本：v1.0*  
*对应代码分支：claude/nostalgic-dirac*
