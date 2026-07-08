# Cloud Run 编排服务模块文档

> **模块路径：** `server/`  
> **部署环境：** Google Cloud Run  
> **技术栈：** Express 5 + TypeScript + Vercel Sandbox SDK + Supabase JS

---

## 目录

1. [模块职责](#1-模块职责)
2. [目录结构](#2-目录结构)
3. [HTTP 服务层](#3-http-服务层)
4. [路由层](#4-路由层)
5. [服务层详解](#5-服务层详解)
   - [sandbox.ts — Vercel Sandbox 生命周期管理](#51-sandboxts--vercel-sandbox-生命周期管理)
   - [extract.ts — PDF 提取服务](#52-extractts--pdf-提取服务)
   - [supabase.ts — 数据库客户端](#53-supabasetsdb-操作)
6. [配置层](#6-配置层)
7. [完整请求处理流程](#7-完整请求处理流程)
8. [错误处理策略](#8-错误处理策略)
9. [关键设计模式](#9-关键设计模式)
10. [依赖清单](#10-依赖清单)

---

## 1. 模块职责

Cloud Run 编排服务是整个系统的**中间层**，承担三个核心职责：

| 职责 | 说明 |
|------|------|
| **任务编排** | 接收前端触发请求，协调 PDF 提取 → 文件下载 → Sandbox 创建 → Agent 启动的完整生命周期 |
| **PDF 提取** | 在 Cloud Run 原生环境（装有 pdftoppm + ImageMagick）中将规划图 PDF 转换为 PNG |
| **长任务托管** | Vercel 函数最多运行 60-300 秒，Cloud Run 可持久运行，托管 10-30 分钟的 Agent 任务 |

**为什么需要 Cloud Run？**

```
前端（Vercel 函数）
    │
    │  问题：Vercel 函数超时 60-300 秒
    │  Agent 运行需要 10-30 分钟
    │
    ▼
Cloud Run（持久化进程）
    │
    │  问题：GCP 负载均衡器空闲 5 分钟后断开连接
    │  
    │  解决：Detached Sandbox 模式 + 轮询等待
    ▼
Vercel Sandbox（Agent 实际执行）
```

---

## 2. 目录结构

```
server/
├── src/
│   ├── index.ts                # Express 入口，端口 8080
│   ├── routes/
│   │   ├── generate.ts         # POST /generate — 触发 Agent 流程
│   │   └── extract.ts          # POST /extract — PDF 提取
│   ├── services/
│   │   ├── sandbox.ts          # Vercel Sandbox 全生命周期管理（848 行）
│   │   ├── supabase.ts         # Supabase 数据库客户端（116 行）
│   │   └── extract.ts          # PDF → PNG 转换（193 行）
│   └── utils/
│       └── config.ts           # 配置、技能路由、提示词构建（334 行）
├── skills/                     # Agent 技能库（随 Sandbox 分发）
│   ├── california-adu/         # 28 个加州法规参考文件
│   ├── adu-corrections-flow/   # 更正分析编排技能
│   ├── adu-corrections-complete/  # 更正回复生成技能
│   ├── adu-city-research/      # 城市法规研究技能
│   ├── adu-plan-review/        # 城市侧审查技能
│   ├── adu-targeted-page-viewer/  # 视觉页面分析技能
│   ├── placentia-adu/          # 城市专项：Placentia
│   └── buena-park-adu/         # 城市专项：Buena Park
└── package.json
```

---

## 3. HTTP 服务层

**文件：** `src/index.ts`

```typescript
// 监听端口（默认 8080，Cloud Run 默认端口）
const PORT = process.env.PORT || 8080

app.use(express.json())        // JSON 请求体解析
app.get('/health', ...)        // 健康检查（Cloud Run 存活探针）
app.use('/generate', generateRouter)
app.use('/extract', extractRouter)

// 全局错误处理
app.use((err, req, res, next) => {
  res.status(500).json({ error: err.message })
})
```

**三个端点：**

| 端点 | 方法 | 返回 |
|------|------|------|
| `/health` | GET | `{ status: 'ok' }` |
| `/generate` | POST | 立即返回 `{ status: 'processing', project_id }` |
| `/extract` | POST | 立即返回 `{ status: 'processing', project_id }` |

所有业务处理均为**异步 fire-and-forget**，路由层立即返回，实际工作在后台进行。

---

## 4. 路由层

### 4.1 POST /generate（`src/routes/generate.ts`）

**请求体 Schema（Zod 校验）：**

```typescript
const GenerateSchema = z.object({
  project_id: z.string().uuid(),
  user_id:    z.string().uuid(),
  flow_type:  z.enum(['city-review', 'corrections-analysis', 'corrections-response'])
})
```

**处理逻辑：**

```
POST /generate
    │
    ├─ Zod 校验请求体
    ├─ 立即返回 202 { status: 'processing', project_id }
    │
    └─ 异步执行 processGeneration()（不等待结果）
         │
         ├─ updateProjectStatus(project_id, 'processing')
         ├─ 如果 flow_type ≠ 'corrections-response':
         │    extractPdfForProject(project_id)  ← PDF 预提取
         ├─ 如果 flow_type = 'corrections-response':
         │    getContractorAnswers(project_id)
         │    getPhase1Outputs(project_id)
         └─ runCrossBeamFlow({
               project_id, user_id, flow_type,
               city, address,
               contractorAnswers?, phase1Artifacts?
            })
```

**关键设计**：路由立即返回，避免 HTTP 连接超时。前端通过 Supabase Realtime 接收后续状态更新。

### 4.2 POST /extract（`src/routes/extract.ts`）

```typescript
const ExtractSchema = z.object({
  project_id: z.string().uuid()
})
```

独立的 PDF 提取触发端点，供前端在上传文件后主动预提取（减少 Agent 启动后的等待时间）。

---

## 5. 服务层详解

### 5.1 `sandbox.ts` — Vercel Sandbox 生命周期管理

这是整个模块最核心的文件（848 行），管理 Vercel Sandbox 从创建到销毁的完整生命周期。

#### 5.1.1 Sandbox 创建

```typescript
async function createSandbox(): Promise<Sandbox> {
  return await Sandbox.create({
    timeout: ms('30m'),    // 30 分钟最长存活时间
    vcpus: 4,              // 4 核 CPU
    runtime: 'node22',     // Node.js 22 运行时
  })
}
```

#### 5.1.2 依赖安装

```typescript
async function installDependencies(sandbox: Sandbox): Promise<void> {
  // 安装 Claude Code CLI（提供 claude_code preset 工具）
  await sandbox.commands.run('npm install -g @anthropic-ai/claude-code')
  
  // 安装 Agent SDK + Supabase 客户端
  await sandbox.commands.run('npm install @anthropic-ai/claude-agent-sdk @supabase/supabase-js')
}
```

#### 5.1.3 项目文件下载

```typescript
async function downloadFilesInSandbox(
  sandbox: Sandbox,
  projectFiles: ProjectFile[]
): Promise<void> {
  // 构建文件清单（Supabase Storage 路径 → 本地路径映射）
  const manifest = projectFiles.map(f => ({
    storagePath: f.storage_path,
    localPath: `/vercel/sandbox/project-files/${f.filename}`,
    bucket: f.storage_path.startsWith('crossbeam-demo-assets/')
      ? 'crossbeam-demo-assets'
      : 'crossbeam-uploads'
  }))
  
  // 在 Sandbox 内执行 Node 脚本下载文件
  const downloadScript = buildDownloadScript(manifest, SUPABASE_URL, SUPABASE_KEY)
  await sandbox.commands.run(`node -e "${downloadScript}"`)
}
```

**文件存储位置：** `/vercel/sandbox/project-files/`

#### 5.1.4 压缩包解压

```typescript
async function unpackArchivesInSandbox(sandbox: Sandbox): Promise<void> {
  // 查找所有 .tar.gz 压缩包（PDF 预提取产物）
  const archives = await sandbox.commands.run(
    'find /vercel/sandbox/project-files -name "*.tar.gz"'
  )
  
  for (const archive of archives) {
    // 解压到同目录，非致命错误（可忽略）
    await sandbox.commands.run(`tar xzf ${archive} -C $(dirname ${archive})`)
    await sandbox.commands.run(`rm ${archive}`)
  }
}
```

#### 5.1.5 技能文件复制

```typescript
async function copySkillsToSandbox(
  sandbox: Sandbox,
  skillNames: string[]   // 动态决定，见 config.ts
): Promise<void> {
  // 从 server/skills/ 读取技能目录
  for (const skillName of skillNames) {
    const skillDir = path.join(__dirname, '../../skills', skillName)
    const targetDir = `/vercel/sandbox/.claude/skills/${skillName}`
    
    // 递归复制到 Sandbox
    await sandbox.files.copy(skillDir, targetDir)
  }
}
```

**技能按需加载：** 不同流程加载不同技能组合（见配置层）。

#### 5.1.6 第一阶段产物预写入（仅 corrections-response）

```typescript
async function writePhase1Artifacts(
  sandbox: Sandbox,
  phase1Artifacts: Phase1Artifacts,
  contractorAnswers: ContractorAnswer[]
): Promise<void> {
  const outputDir = '/vercel/sandbox/project-files/output'
  await sandbox.commands.run(`mkdir -p ${outputDir}`)
  
  // 写入 Phase 1 分析产物
  for (const [filename, content] of Object.entries(phase1Artifacts)) {
    await sandbox.files.write(`${outputDir}/${filename}`, JSON.stringify(content))
  }
  
  // 写入承包商答案
  await sandbox.files.write(
    `${outputDir}/contractor_answers.json`,
    JSON.stringify(contractorAnswers)
  )
}
```

#### 5.1.7 Agent 启动（核心）

这是整个模块最关键的函数，以 **detached 模式**运行 Agent。

```typescript
async function runAgent(
  sandbox: Sandbox,
  opts: AgentRunOptions
): Promise<void> {
  const { project_id, user_id, flow_type, city, address, budget } = opts
  
  // 构建 Agent 执行脚本
  const agentScript = buildAgentScript({
    project_id,
    user_id,
    flow_type,
    prompt: buildPrompt(flow_type, city, address, ...),
    systemAppend: getSystemAppend(flow_type),
    model: CONFIG.MODEL,               // 'claude-opus-4-6'
    maxTurns: budget.maxTurns,         // 500
    maxBudgetUsd: budget.maxBudgetUsd, // $50
    supabaseUrl: SUPABASE_URL,
    supabaseKey: SUPABASE_SERVICE_ROLE_KEY,
  })
  
  // 将脚本写入 Sandbox
  await sandbox.files.write('/vercel/sandbox/run-agent.mjs', agentScript)
  
  // ★ 以 DETACHED 模式启动（关键：不等待进程结束）
  await sandbox.commands.run(
    'node /vercel/sandbox/run-agent.mjs > /vercel/sandbox/agent.log 2>&1 &'
  )
  
  // ★ 轮询等待（最多 120 次 × 30 秒 = 60 分钟）
  await pollUntilComplete(sandbox, project_id)
}
```

**Agent 脚本内部结构：**

```javascript
// run-agent.mjs（在 Sandbox 内执行）
import { query } from '@anthropic-ai/claude-agent-sdk'
import { createClient } from '@supabase/supabase-js'

const supabase = createClient(SUPABASE_URL, SUPABASE_KEY)

// 1. 运行 Agent（阻塞直到完成）
const result = await query({
  prompt: ORCHESTRATOR_PROMPT,
  options: {
    model: 'claude-opus-4-6',
    permissionMode: 'bypassPermissions',
    preset: 'claude_code',              // 启用 Bash/文件 I/O/网络工具
    maxTurns: 500,
    maxBudgetUsd: 50,
    cwd: '/vercel/sandbox',
    systemPromptAppend: SYSTEM_APPEND,
  }
})

// 2. 读取 Agent 产物（跳过二进制文件）
const outputFiles = readOutputFiles('/vercel/sandbox/project-files/output')

// 3. 提取 contractor_questions.json（如果存在）
const questions = parseContractorQuestions(outputFiles)

// 4. 写入 Supabase
await supabase.from('outputs').insert({
  project_id,
  flow_phase,
  version,             // 按 project_id + flow_phase 自动递增
  corrections_analysis_json: outputFiles['corrections_categorized.json'],
  contractor_questions_json: outputFiles['contractor_questions.json'],
  response_letter_md: outputFiles['response_letter.md'],
  // ...其他字段
  agent_cost_usd: result.cost,
  agent_turns: result.turns,
  agent_duration_ms: result.duration,
})

// 5. 写入承包商问题（逐条插入 contractor_answers 表）
for (const question of questions) {
  await supabase.from('contractor_answers').insert({ project_id, ...question })
}

// 6. 更新项目状态
await supabase.from('projects').update({
  status: flow_type === 'corrections-analysis' ? 'awaiting-answers' : 'completed'
})
```

#### 5.1.8 Detached 模式轮询等待

```typescript
async function pollUntilComplete(
  sandbox: Sandbox,
  project_id: string
): Promise<void> {
  const MAX_ATTEMPTS = 120    // 最多等待 60 分钟
  const INTERVAL_MS = 30_000  // 每 30 秒检查一次
  
  for (let attempt = 0; attempt < MAX_ATTEMPTS; attempt++) {
    await sleep(INTERVAL_MS)
    
    // 检查项目状态（Agent 完成后会写 Supabase）
    const project = await getProject(project_id)
    
    if (['completed', 'failed', 'awaiting-answers'].includes(project.status)) {
      return  // Agent 已完成
    }
    
    // 检查 Sandbox 是否仍然存活
    const sandboxStatus = await sandbox.getStatus()
    if (sandboxStatus === 'stopped') {
      throw new Error('Sandbox stopped unexpectedly')
    }
  }
  
  throw new Error('Agent timed out after 60 minutes')
}
```

**为什么需要 Detached 模式？**

```
不使用 Detached 模式的问题：
  Cloud Run ─────────────────────────────→ Sandbox（等待 Agent 完成）
       │                                        │
       │  GCP 负载均衡器在 5 分钟空闲后断开连接   │
       ×                                        │
  连接断开！Cloud Run 进程异常退出               │ Agent 继续运行
  → Sandbox 被垃圾回收                          │（但无人收集结果）
  
使用 Detached 模式的流程：
  Cloud Run ─→ 启动 Agent（detached）─→ 开始轮询（每 30s 查一次 DB）
       │                                        │
       │  GCP 负载均衡器断开？没关系              │
       │  Cloud Run 进程仍在运行                 │ Agent 继续运行
       │  轮询继续...                            │
       └────────────── 30 秒后再次轮询 ──────────┘
```

#### 5.1.9 主编排函数

```typescript
async function runCrossBeamFlow(opts: FlowOptions): Promise<void> {
  const { project_id, flow_type, city } = opts
  let sandbox: Sandbox | null = null
  
  try {
    // Step 1: 创建 Sandbox
    insertMessage(project_id, 'system', '正在创建执行环境...')
    sandbox = await createSandbox()
    
    // Step 2: 安装依赖
    insertMessage(project_id, 'system', '正在安装依赖...')
    await installDependencies(sandbox)
    
    // Step 3: 下载项目文件
    insertMessage(project_id, 'system', '正在下载项目文件...')
    const files = await getProjectFiles(project_id)
    await downloadFilesInSandbox(sandbox, files)
    
    // Step 4: 解压预提取产物
    await unpackArchivesInSandbox(sandbox)
    
    // Step 5: 复制技能文件
    insertMessage(project_id, 'system', '正在加载知识库...')
    const skills = getFlowSkills(flow_type, city)
    await copySkillsToSandbox(sandbox, skills)
    
    // Step 6: 写入 Phase 1 产物（仅 corrections-response）
    if (flow_type === 'corrections-response' && opts.phase1Artifacts) {
      await writePhase1Artifacts(sandbox, opts.phase1Artifacts, opts.contractorAnswers)
    }
    
    // Step 7: 启动 Agent（Detached 模式，内部轮询等待）
    insertMessage(project_id, 'system', '正在启动 AI 分析...')
    await runAgent(sandbox, { ...opts, budget: FLOW_BUDGET[flow_type] })
    
  } catch (error) {
    // 更新项目状态为失败
    await updateProjectStatus(project_id, 'failed', error.message)
    insertMessage(project_id, 'system', `分析失败：${error.message}`)
  } finally {
    // 始终清理 Sandbox（无论成功还是失败）
    if (sandbox) await sandbox.stop()
  }
}
```

---

### 5.2 `extract.ts` — PDF 提取服务

**文件：** `src/services/extract.ts`

Cloud Run 原生环境安装了 pdftoppm（poppler）和 ImageMagick，直接在本地执行图像转换，无需 Sandbox。

#### 完整提取流程

```
PDF 文件（Supabase Storage）
    │
    ▼
下载到 /tmp/cb-extract-XXXX/
    │
    ▼
pdftoppm（200 DPI 渲染）
    │  每页 PDF → page-01.png, page-02.png, ...
    │  超时：120 秒
    ▼
ImageMagick crop（标题块裁剪）
    │  裁剪每页右下角 25% × 35%
    │  → title-block-01.png, title-block-02.png, ...
    │  超时：30 秒/张
    ▼
tar 压缩打包
    │  pages-png.tar.gz      ← 所有页面 PNG
    │  title-blocks.tar.gz   ← 所有标题块 PNG
    ▼
上传到 Supabase Storage
    │  路径：crossbeam-uploads/{user_id}/{project_id}/
    ▼
写入 files 表（mime_type = 'application/gzip'）
```

**关键实现：**

```typescript
async function extractPdfForProject(projectId: string): Promise<void> {
  // 1. 检查是否已提取（幂等）
  const existingArchives = await checkExistingArchives(projectId)
  if (existingArchives.length >= 2) return  // 已有 pages + title-blocks
  
  // 2. 下载 PDF
  const pdfFile = await getPdfFile(projectId)
  const tmpDir = `/tmp/cb-extract-${Date.now()}`
  await fs.mkdir(tmpDir)
  await downloadFile(pdfFile.storage_path, `${tmpDir}/input.pdf`)
  
  // 3. pdftoppm 渲染
  await execTimeout(
    `pdftoppm -r 200 -png ${tmpDir}/input.pdf ${tmpDir}/page`,
    120_000  // 2 分钟超时
  )
  
  // 重命名为零填充格式：page-01.png, page-02.png, ...
  await renamePages(tmpDir)
  
  // 4. ImageMagick 裁剪标题块
  const pages = await listPages(tmpDir)
  for (const page of pages) {
    const w = getWidth(page), h = getHeight(page)
    await execTimeout(
      // 右下角 25% 宽 × 35% 高
      `magick ${page} -crop ${Math.round(w*0.25)}x${Math.round(h*0.35)}+${Math.round(w*0.75)}+${Math.round(h*0.65)} title-${page}`,
      30_000
    )
  }
  
  // 5. 打包
  await execTimeout(`tar czf ${tmpDir}/pages-png.tar.gz -C ${tmpDir} page-*.png`, 60_000)
  await execTimeout(`tar czf ${tmpDir}/title-blocks.tar.gz -C ${tmpDir} title-*.png`, 60_000)
  
  // 6. 上传 + 写 DB
  await uploadAndRecord(projectId, `${tmpDir}/pages-png.tar.gz`, 'pages-png.tar.gz')
  await uploadAndRecord(projectId, `${tmpDir}/title-blocks.tar.gz`, 'title-blocks.tar.gz')
  
  // 7. 清理临时目录
  await fs.rm(tmpDir, { recursive: true })
}
```

**为什么在 Cloud Run 而不是 Sandbox 提取？**

| 方案 | 问题 |
|------|------|
| 在 Sandbox 内提取 | Agent token 消耗在文件操作上，而非 AI 推理 |
| 在 Cloud Run 提取 | 原生 Linux 环境，速度快（26 页约 30-60 秒），零 token 消耗 |

---

### 5.3 `supabase.ts` — DB 操作

**文件：** `src/services/supabase.ts`

封装所有 Supabase 数据库操作，使用 Service Role Key（绕过 RLS）。

```typescript
// 使用 Service Role Key，绕过 Row Level Security
const supabase = createClient(SUPABASE_URL, SUPABASE_SERVICE_ROLE_KEY)
```

**函数清单：**

| 函数 | 操作 | 说明 |
|------|------|------|
| `updateProjectStatus(id, status, errorMsg?)` | UPDATE projects | 更新状态 + updated_at |
| `getProject(id)` | SELECT projects | 获取完整项目记录（含 city, address）|
| `getProjectFiles(id)` | SELECT files | 获取项目所有文件（含 storage_path）|
| `getContractorAnswers(id)` | SELECT contractor_answers | 按 created_at 排序 |
| `getPhase1Outputs(id)` | SELECT outputs | flow_phase='analysis' 最新版本 |
| `insertMessage(id, role, content)` | INSERT messages | fire-and-forget，不等待 |

**insertMessage 的 fire-and-forget 设计：**

```typescript
function insertMessage(projectId: string, role: string, content: string): void {
  // 注意：没有 await，不阻塞主流程
  supabase.from('messages').insert({ project_id: projectId, role, content })
    .then(() => {})
    .catch(err => console.error('Message insert failed:', err))
}
```

Realtime 订阅会立即将新消息推送到前端，实现流式状态更新效果。

---

## 6. 配置层

**文件：** `src/utils/config.ts`

### 6.1 Sandbox 基础配置

```typescript
export const CONFIG = {
  SANDBOX_TIMEOUT: ms('30m'),   // 30 分钟
  SANDBOX_VCPUS: 4,
  RUNTIME: 'node22',
  MODEL: 'claude-opus-4-6',
  
  // Sandbox 内路径
  PROJECT_FILES_DIR: '/vercel/sandbox/project-files',
  OUTPUT_DIR: '/vercel/sandbox/project-files/output',
  SKILLS_DIR: '/vercel/sandbox/.claude/skills',
}
```

### 6.2 流程预算

```typescript
export const FLOW_BUDGET: Record<FlowType, Budget> = {
  'city-review':            { maxTurns: 500, maxBudgetUsd: 50 },
  'corrections-analysis':   { maxTurns: 500, maxBudgetUsd: 50 },
  'corrections-response':   { maxTurns: 150, maxBudgetUsd: 20 },
}
```

### 6.3 城市专项技能路由

```typescript
// 已接入城市（离线法规数据，响应更快更准确）
const ONBOARDED_CITIES: Record<string, string> = {
  'placentia':  'placentia-adu',
  'buena-park': 'buena-park-adu',
}

export function citySlug(city: string): string {
  return city.toLowerCase().replace(/\s+/g, '-').trim()
}

export function isCityOnboarded(city: string): boolean {
  return citySlug(city) in ONBOARDED_CITIES
}

export function getCitySkillName(city: string): string | null {
  return ONBOARDED_CITIES[citySlug(city)] ?? null
}
```

### 6.4 动态技能选择

```typescript
export function getFlowSkills(flowType: FlowType, city?: string): string[] {
  const citySkill = city ? getCitySkillName(city) : null
  
  switch (flowType) {
    case 'city-review':
      return [
        'california-adu',
        'adu-plan-review',
        'adu-targeted-page-viewer',
        ...(citySkill ? [citySkill] : []),  // 有专项技能则用专项
      ]
      
    case 'corrections-analysis':
      return [
        'california-adu',
        'adu-corrections-flow',
        'adu-targeted-page-viewer',
        ...(citySkill ? [citySkill] : ['adu-city-research']),  // 无专项则用网络研究
      ]
      
    case 'corrections-response':
      return [
        'california-adu',
        'adu-corrections-complete',
        ...(citySkill ? [citySkill] : []),
        // ★ 不加载 adu-city-research 或 adu-targeted-page-viewer
        //   Phase 2 不需要网络研究，只需生成回复
      ]
  }
}
```

### 6.5 提示词构建

```typescript
export function buildPrompt(
  flowType: FlowType,
  city: string,
  address: string,
  contractorAnswersJson?: string,
  preExtracted?: boolean
): string {
  switch (flowType) {
    case 'corrections-analysis':
      return `
你是一位专业的 ADU 许可证顾问。
项目地址：${address}，城市：${city}

项目文件已下载到 /vercel/sandbox/project-files/
${preExtracted ? '规划图已预提取为 PNG，位于 pages-png/ 目录' : '需要自行提取 PDF'}

请使用 adu-corrections-flow 技能，执行完整的更正分析流程。
      `.trim()
      
    case 'corrections-response':
      return `
你是一位专业的 ADU 许可证顾问。
Phase 1 分析已完成，承包商答案如下：
${contractorAnswersJson}

请使用 adu-corrections-complete 技能，基于分析结果和承包商答案，
生成完整的回复包（回复信 + 专业范围 + 更正报告 + 图纸注释）。
      `.trim()
    
    // ...
  }
}
```

---

## 7. 完整请求处理流程

以 `corrections-analysis` 为例，端到端时序图：

```
前端                Cloud Run              Supabase              Vercel Sandbox
  │                     │                     │                        │
  │ POST /generate      │                     │                        │
  │────────────────────>│                     │                        │
  │                     │                     │                        │
  │ 202 { processing }  │                     │                        │
  │<────────────────────│                     │                        │
  │                     │                     │                        │
  │                     │ UPDATE status='processing'                   │
  │                     │────────────────────>│                        │
  │                     │                     │                        │
  │  ← Realtime 推送 ──│─────────────────────│                        │
  │                     │                     │                        │
  │                     │ pdftoppm + ImageMagick（本地，约 30-60s）     │
  │                     │───────── PDF 转 PNG ─────────────────────────│（上传产物到 Storage）
  │                     │                     │                        │
  │                     │ INSERT message '正在创建执行环境'              │
  │                     │────────────────────>│                        │
  │  ← Realtime 推送 ──│─────────────────────│                        │
  │                     │                     │                        │
  │                     │ createSandbox()     │                        │
  │                     │──────────────────────────────────────────────>（创建 Sandbox）
  │                     │                     │                        │
  │                     │ installDependencies()                        │
  │                     │──────────────────────────────────────────────>（npm install）
  │                     │                     │                        │
  │                     │ downloadFilesInSandbox()                     │
  │                     │──────────────────────────────────────────────>（下载 PDF + PNG 包）
  │                     │                     │                        │
  │                     │ unpackArchivesInSandbox()                    │
  │                     │──────────────────────────────────────────────>（tar 解压）
  │                     │                     │                        │
  │                     │ copySkillsToSandbox()                        │
  │                     │──────────────────────────────────────────────>（复制技能文件）
  │                     │                     │                        │
  │                     │ runAgent()（detached 启动）                   │
  │                     │──────────────────────────────────────────────>（node run-agent.mjs &）
  │                     │                     │                        │
  │                     │    轮询 30s...       │                        │ Agent 运行中...
  │                     │<─ ─ ─ ─ ─ ─ ─ ─ ─ ─│─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─│ INSERT messages（Realtime）
  │  ← Realtime 推送 ──│─────────────────────│                        │ 
  │                     │                     │          ← 约 10-15 分钟后 →
  │                     │                     │                        │
  │                     │                     │  INSERT outputs{...}   │
  │                     │                     │<───────────────────────│
  │                     │                     │                        │
  │                     │                     │  UPDATE status='awaiting-answers'
  │                     │                     │<───────────────────────│
  │                     │                     │                        │
  │  ← Realtime 推送 ──│─────────────────────│                        │
  │（UI 更新：显示问答表单）                   │                        │
  │                     │                     │                        │
  │                     │ pollUntilComplete 检测到状态变化，退出轮询     │
  │                     │ sandbox.stop()      │                        │
  │                     │──────────────────────────────────────────────>（销毁 Sandbox）
```

---

## 8. 错误处理策略

| 错误类型 | 处理方式 |
|---------|---------|
| Zod 校验失败 | 400 响应，立即返回 |
| PDF 提取失败 | 非致命，记录日志，Agent 可在 Sandbox 内重新提取 |
| Sandbox 创建失败 | 更新 project.status='failed'，写 error_message |
| Agent 执行失败 | Sandbox 内脚本捕获，写 project.status='failed' |
| Agent 超时（60 分钟）| 更新 status='failed'，停止 Sandbox |
| Sandbox 意外停止 | 轮询检测到，更新 status='failed' |
| 消息写入失败 | fire-and-forget，静默忽略（不影响主流程）|

```typescript
// 在 runCrossBeamFlow 中
try {
  // ... 主流程
} catch (error) {
  await updateProjectStatus(project_id, 'failed', error.message)
  insertMessage(project_id, 'system', `错误：${error.message}`)
} finally {
  if (sandbox) {
    try { await sandbox.stop() } catch (_) {}  // 确保 Sandbox 被销毁
  }
}
```

---

## 9. 关键设计模式

### 9.1 异步 Fire-and-Forget

```
HTTP 请求 → 立即 202 响应 → 后台异步处理
```
避免 HTTP 连接超时，前端通过 Realtime 获取进度。

### 9.2 Detached Sandbox + 轮询

```
Cloud Run 启动 Agent（&）→ 每 30 秒查询 DB 状态
GCP 连接断开？→ Cloud Run 进程继续运行，轮询不中断
Agent 完成 → 写 DB → 轮询检测到 → 清理 Sandbox
```

### 9.3 幂等性设计

- PDF 提取前检查是否已存在产物（避免重复提取）
- Sandbox 内解压使用 `-f` 覆盖标志
- Agent 输出版本号自动递增（允许重跑）

### 9.4 基于文件的子 Agent 通信

```
主 Agent（编排者）
    │
    ├─ 启动子 Agent A → 子 Agent 将结果写入 findings-a.json
    ├─ 启动子 Agent B → 子 Agent 将结果写入 findings-b.json
    └─ 等待完成 → 主 Agent 只读取文件路径，不读取内容
                  → 避免将大量 JSON 加载到主 Agent 上下文
```

---

## 10. 依赖清单

```json
{
  "dependencies": {
    "@supabase/supabase-js": "^2.49.0",  // Supabase 数据库 + Storage 客户端
    "@vercel/sandbox": "^1.1.0",         // Vercel Sandbox 生命周期管理
    "express": "^5.0.1",                 // HTTP 服务器
    "ms": "^2.1.3",                      // 时间字符串解析（'30m' → 毫秒）
    "zod": "^3.24.0"                     // 请求体 Schema 校验
  },
  "devDependencies": {
    "@types/express": "^5.0.0",
    "@types/ms": "^0.7.34",
    "@types/node": "^22.10.2",
    "tsx": "^4.19.0",     // TypeScript 直接执行（无需编译）
    "typescript": "^5.7.0"
  }
}
```

**注意：** 生产部署的 Cloud Run 镜像还需要系统依赖：
- `poppler-utils`（提供 pdftoppm）
- `imagemagick`（提供 convert/magick）

---

*文档版本：v1.0 | 2026-05-11*  
*对应源码：`server/` 目录*
