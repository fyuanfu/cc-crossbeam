# Agent 多运行时并行执行机制

> **核心问题**：如何在单个 LLM 对话上下文内，安全地并行运行多个专业子 Agent，同时避免 token 膨胀、图片堆积和上下文污染？
>
> **核心答案**：**文件系统作为共享内存 + Task 工具管理子 Agent + 滚动窗口限制并发数**

---

## 目录

1. [并行执行的核心挑战](#1-并行执行的核心挑战)
2. [解决方案：文件系统中间层](#2-解决方案文件系统中间层)
3. [并行实现机制：Task 工具](#3-并行实现机制task-工具)
4. [滚动并发窗口](#4-滚动并发窗口)
5. [三大并行场景详解](#5-三大并行场景详解)
   - [场景A：更正分析 Phase 3（状态法 + 城市法 + 图纸视觉）](#场景a更正分析-phase-3三路并行)
   - [场景B：城市审查 Phase 2（五专业图纸审查）](#场景b城市审查-phase-2五专业滚动审查)
   - [场景C：城市法规提取 Fan-out 模式](#场景c城市法规-fan-out-模式)
6. [子 Agent 上下文隔离](#6-子-agent-上下文隔离)
7. [信息聚合：Phase 4 合并子 Agent](#7-信息聚合phase-4-合并子-agent)
8. [时序图：完整并行执行流程](#8-时序图完整并行执行流程)
9. [并行参数与预算控制](#9-并行参数与预算控制)
10. [与外层并行的区别](#10-与外层并行的区别)

---

## 1. 并行执行的核心挑战

### 挑战 1：图片 Token 堆积

规划图每页是 **7,400px × 4,000px+** 的高分辨率 PNG。若主 Agent 直接读取 26 页图纸：

```
主 Agent 上下文（❌ 错误做法）：
  [系统提示] + [对话历史] + [图片 1] + [图片 2] + ... + [图片 26]
                                                              ↑
                                          每张图片约 800-1500 token
                                          26 张 × 1200 token = 31,200 token
                                          → 超出上下文限制，推理质量下降
```

### 挑战 2：专业知识污染

一个 Agent 同时查询州法规、城市法规、图纸观察，上下文中混入大量不相关信息，降低推理准确性。

### 挑战 3：串行速度慢

14 条更正项，每项需要：州法查询（60s）+ 城市法查询（90s）+ 图纸分析（30s）= 180s  
串行总计：14 × 180s = **42 分钟**

---

## 2. 解决方案：文件系统中间层

**核心思想：子 Agent 不向主 Agent 返回大型数据，而是写入共享文件系统。**

```
主 Agent（编排者）
    │
    │  只传递：任务描述 + 输出文件路径
    ▼
子 Agent A ──→ 写入 findings-a.json
子 Agent B ──→ 写入 findings-b.json   } 文件系统作为共享内存
子 Agent C ──→ 写入 findings-c.json
    │
    │  主 Agent 只验证文件是否存在（Glob），不读取内容
    ▼
Phase 4 合并子 Agent（专用角色）──→ 读取所有文件 ──→ 生成最终输出
```

**主 Agent 视角的子 Agent 调用（伪代码）：**

```
// 主 Agent 启动并发子 Agent
tasks = [
  Task("查询加州 ADU 法规，输出到 state_law_findings.json"),
  Task("搜索 Placentia 城市法规，输出到 city_discovery.json"),
  Task("分析 Sheet A3 规划图页面，输出到 sheet_observations.json"),
]

// 并发等待
await all(tasks)

// 验证产物（不读取内容）
assert Glob("project-files/output/state_law_findings.json").exists()
assert Glob("project-files/output/city_discovery.json").exists()
assert Glob("project-files/output/sheet_observations.json").exists()

// 主 Agent 仅记录："所有子 Agent 完成，产物已就绪"
```

---

## 3. 并行实现机制：Task 工具

Agent SDK 的 `claude_code` preset 提供 **Task 工具**，用于在 Vercel Sandbox 内启动独立的子 Agent 进程。

### Task 工具的工作原理

```
主 Agent（claude-opus-4-6）
    │
    │  调用 Task 工具（类似线程 spawn）
    │  Task({
    │    description: "研究加州 ADU 法规",
    │    prompt: "你是州法规研究员，...",
    │    tools: ['Skill', 'Read', 'Glob', 'Write'],
    │  })
    │
    ▼
子 Agent 进程（独立上下文）
    ├─ 自己的对话历史（不继承主 Agent）
    ├─ 自己的工具调用记录
    └─ 独立的 token 预算
    
    子 Agent 完成 → 将结果写入文件 → 退出
    主 Agent 的 Task 工具调用返回：简短摘要
```

**关键特性：**

| 特性 | 说明 |
|------|------|
| **上下文隔离** | 子 Agent 有独立的对话历史，不污染主 Agent |
| **并发执行** | 多个 Task 调用可以同时进行（非阻塞）|
| **结果通信** | 子 Agent 通过写文件传递数据，不在对话中返回大数据 |
| **工具限制** | 可为子 Agent 指定受限工具集（如只给文件操作工具）|

### 主 Agent 如何并发启动多个 Task

```
# 主 Agent 的对话（概念示意）

Turn 15：
  我需要并行研究三个方向：
  1. 州法规
  2. 城市法规  
  3. 图纸观察
  
  → 同时调用三个 Task（不等待第一个完成就调用第二个）

[Task A 开始执行...]   ← 子进程 1
[Task B 开始执行...]   ← 子进程 2
[Task C 开始执行...]   ← 子进程 3

Turn 16（等待所有 Task 完成）：
  Task A 完成：已将结果写入 state_law_findings.json
  Task B 完成：已将结果写入 city_discovery.json
  Task C 完成：已将结果写入 sheet_observations.json
  
  → 三个子 Agent 的上下文已全部释放
  → 主 Agent 上下文中只有简短的完成确认
```

---

## 4. 滚动并发窗口

**核心设计：不是"所有子 Agent 同时启动"，而是"最多 N 个并发"。**

### 为什么需要滚动窗口？

```
14 条更正项，若全部同时启动：

错误做法（全并发）：
  [Task 1][Task 2][Task 3]...[Task 14] 同时启动
  ↓
  Claude API 并发请求过多 → 速率限制（429）
  + 主 Agent 等待 14 个 Task 的消息 → 上下文膨胀

正确做法（滚动窗口，N=3）：
  批次 1：[Task 1][Task 2][Task 3]  ← 同时运行
  等待完成...
  批次 2：[Task 4][Task 5][Task 6]  ← 同时运行
  等待完成...
  批次 3：[Task 7][Task 8][Task 9]  ← 同时运行
  ...
```

### 滚动窗口执行时序

```
时间轴（每格 = 约 30 秒）

窗口大小 = 3，共 14 条更正项

t=0  ├─[Item 1]─────────────────────────────────────────┤
     ├─[Item 2]────────────────────────────────────────┤
     ├─[Item 3]──────────────────────────────────────────┤
                                                         ↑ 批次 1 完成
t=90 ├─[Item 4]─────────────────────────┤
     ├─[Item 5]──────────────────────────────┤
     ├─[Item 6]───────────────────────────┤
                                            ↑ 批次 2 完成
t=180 ├─[Item 7]────────────────────────────────┤
      ├─[Item 8]──────────────────────────────────┤
      ├─[Item 9]─────────────────────────────────┤
                                                   ↑ 批次 3 完成
... 以此类推

总计：约 5 批次 × 90 秒 = 7.5 分钟
对比串行：14 × 90 秒 = 21 分钟
提速：约 2.8× 
```

### adu-corrections-flow 技能中的滚动窗口说明

```markdown
# 来自 server/skills/adu-corrections-flow/SKILL.md

## Phase 3C — Sheet Viewer（图纸观察子 Agent）

对于每条引用图纸的更正项：
- 启动一个 adu-targeted-page-viewer 子 Agent
- 子 Agent 任务：读取指定图纸页面 → 描述现状 → 写入 sheet_observations.json
- 滚动窗口：最多 3 个并发
- 每个子 Agent 完成后立即释放，主 Agent 不在上下文中保留图片
```

---

## 5. 三大并行场景详解

### 场景A：更正分析 Phase 3（三路并行）

**来源：** `server/skills/adu-corrections-flow/SKILL.md`

这是系统中**最复杂的并行场景**，Phase 3 同时启动三个方向的研究：

```
Phase 2 完成（sheet-manifest.json 已就绪）
    │
    ├─────────────────────────────────────┐
    │                                     │
    ▼                                     │
Phase 3A                Phase 3B          Phase 3C
州法规研究              城市发现            图纸观察
    │                     │                  │
    │ 加载 california-adu  │ 使用 WebSearch   │ 使用 adu-targeted-
    │ 技能（28 个参考文件）│ 搜索城市 ADU 页面│ page-viewer 技能
    │                     │                  │
    │ 查询所有更正项引用   │ 仅 WebSearch     │ 对每条引用图纸的
    │ 的法规条款          │（不做 WebFetch）  │ 更正项，读取对应
    │                     │                  │ 图纸页面
    │ 去重：若 2 条引用    │                  │
    │ 同一条款，只查一次  │                  │ 滚动窗口 × 3
    │                     │                  │
    ▼                     ▼                  ▼
state_law_findings.json  city_discovery.json  sheet_observations.json
（约 60s）              （约 30s）           （约 60s）
    │                     │                  │
    └─────────────────────┴──────────────────┘
                          │
                          ▼
                    Phase 3.5：城市内容提取
                    （顺序，使用 Phase 3B 发现的 URL）
                    WebFetch 提取城市法规全文
                    → city_research_findings.json（约 60-90s）
                          │
                          ▼
                    Phase 4：合并分析（专用子 Agent）
```

**三路并行的关键约束：**

| 子 Agent | 工具权限 | 禁止 |
|---------|---------|------|
| 3A 州法规 | `Skill`(california-adu), `Read`, `Glob`, `Write` | 不做 WebSearch/WebFetch |
| 3B 城市发现 | `WebSearch`, `Write` | 不做 WebFetch（Phase 3.5 才做）|
| 3C 图纸观察 | `Skill`(page-viewer), `Read`, `Write`, `Task` | 不做法规查询 |

**时间节省：**
- 串行：60 + 30 + 60 = 150 秒
- 并行：max(60, 30, 60) = **60 秒**
- 节省：60%

---

### 场景B：城市审查 Phase 2（五专业滚动审查）

**来源：** `server/skills/adu-plan-review/SKILL.md`

城市侧审查需要五个专业同时检查不同图纸，使用滚动窗口（N=3）：

```
sheet-manifest.json（已知各图纸）
    │
    ▼
五专业子 Agent（滚动窗口 × 3）：

批次 1（并发）：
  ┌──────────────────────────────────┐
  │ arch-a（建筑-A）                  │
  │ 负责：封面 + 平面图               │
  │ 图纸：C1, A1, A2                 │
  │ 输出：findings-arch-a.json        │
  └──────────────────────────────────┘
  ┌──────────────────────────────────┐
  │ arch-b（建筑-B）                  │
  │ 负责：立面图 + 屋顶图 + 剖面       │
  │ 图纸：A3, A4, A5                 │
  │ 输出：findings-arch-b.json        │
  └──────────────────────────────────┘
  ┌──────────────────────────────────┐
  │ site-civil（场地 + 市政）          │
  │ 负责：场地图 + 排水 + 管线         │
  │ 图纸：C2, C3                     │
  │ 输出：findings-site-civil.json    │
  └──────────────────────────────────┘
  
批次 1 完成（约 90-150 秒）

批次 2（并发）：
  ┌──────────────────────────────────┐
  │ structural（结构）                │
  │ 负责：基础 + 框架 + 细节           │
  │ 图纸：S1, S2, S3                 │
  │ 输出：findings-structural.json    │
  └──────────────────────────────────┘
  ┌──────────────────────────────────┐
  │ mep-energy（机电 + 能源）          │
  │ 负责：给排水 + 暖通 + 电气 + Title24│
  │ 图纸：M1, E1, E2                 │
  │ 输出：findings-mep-energy.json    │
  └──────────────────────────────────┘

批次 2 完成（约 90-150 秒）
```

**每个专业子 Agent 的输出结构：**

```json
// 以 findings-arch-a.json 为例
{
  "discipline": "arch-a",
  "findings": [
    {
      "check_id": "1A",
      "sheet_id": "A1",
      "status": "FAIL",
      "visual_confidence": "HIGH",
      "observation": "平面图未标注 ADU 与主宅之间的防火间距，仅显示 2 英尺",
      "code_ref": "CRC R302.1"
    },
    {
      "check_id": "2B", 
      "sheet_id": "A2",
      "status": "PASS",
      "visual_confidence": "HIGH",
      "observation": "厨房最小净宽 7 英尺，符合 CRC 标准",
      "code_ref": "CRC R304"
    }
  ]
}
```

**主 Agent 对子 Agent 的要求：**

```markdown
# 来自 adu-plan-review/SKILL.md

每个审查子 Agent 必须：
1. 读取分配的图纸页面（PNG）
2. 将发现写入 findings-{discipline}.json
3. 返回 SHORT SUMMARY ONLY（不超过 5 行）

❌ 禁止在返回文本中包含完整 JSON
❌ 禁止在返回文本中复述所有发现
✅ 仅返回："已完成审查 X 张图纸，发现 Y 个问题，结果写入 findings-arch-a.json"
```

这确保**主 Agent 的上下文只累积简短摘要**，而非 26 页图纸的完整分析。

---

### 场景C：城市法规 Fan-out 模式

**来源：** `server/skills/adu-corrections-flow/SKILL.md`，Phase 3.5 扩展

当城市法规需要提取的 URL 超过 6 个时，启动 Fan-out 模式：

```
Phase 3B 发现 8 个 URL：
  url_1（ADU 规范页）
  url_2（停车要求页）
  url_3（退界要求页）
  url_4（标准细节页）
  url_5（信息公告 1）
  url_6（信息公告 2）
  url_7（申报要求页）
  url_8（审查清单页）

Fan-out：按类别分组，3 个子 Agent 并发

┌─────────────────────────┐ ┌──────────────────────────┐ ┌──────────────────────────┐
│ Subagent 1              │ │ Subagent 2               │ │ Subagent 3               │
│ url_1, url_2, url_3     │ │ url_4, url_5, url_6      │ │ url_7, url_8             │
│ （规范 + 退界 + 停车）  │ │ （标准细节 + 信息公告）  │ │ （申报 + 审查清单）       │
│ → city_extract_1.json   │ │ → city_extract_2.json    │ │ → city_extract_3.json    │
└─────────────────────────┘ └──────────────────────────┘ └──────────────────────────┘
          │                           │                           │
          └───────────────────────────┴───────────────────────────┘
                                      │
                                      ▼
                            主 Agent 合并三个文件
                            → city_research_findings.json
```

**触发条件：**

| URL 数量 | 策略 |
|---------|------|
| ≤ 3 个 | 单 Agent 顺序 WebFetch |
| 4-5 个 | 单 Agent 顺序 WebFetch（仍可接受）|
| ≥ 6 个 | Fan-out 模式（2-3 个并发子 Agent）|

---

## 6. 子 Agent 上下文隔离

### 为什么图片不会在主 Agent 中累积？

```
❌ 错误理解（图片累积）：
主 Agent 对话：
  Turn 1: 读图片 page-01.png → 图片数据进入上下文
  Turn 2: 读图片 page-03.png → 图片数据进入上下文
  Turn 3: 读图片 page-07.png → 图片数据进入上下文
  ...26 张 → 上下文爆炸

✅ 正确实现（隔离读取）：
主 Agent 对话：
  Turn 1: Task("读取 page-01.png，输出到 obs-01.json")
           → 子 Agent 1 启动：独立上下文，读图片，写文件，退出
           → 主 Agent 收到："完成，结果在 obs-01.json"
  Turn 2: Task("读取 page-03.png，输出到 obs-03.json")
           → 子 Agent 2 启动：独立上下文，读图片，写文件，退出
           → 主 Agent 收到："完成，结果在 obs-03.json"
  
  主 Agent 上下文中：永远只有简短的任务结果描述，无图片数据
```

### 子 Agent 生命周期

```
创建      运行                    完成
  │         │                      │
  ▼         ▼                      ▼
[独立上下文] ─→ [工具调用 + 推理] ─→ [写文件] ─→ [上下文销毁]
                                              ↑
                                    上下文内容完全释放
                                    包括读取的图片数据
```

---

## 7. 信息聚合：Phase 4 合并子 Agent

所有并行子 Agent 完成后，启动一个**专用的合并子 Agent**（不是主 Agent 直接读取）。

### 为什么需要专用合并子 Agent？

```
主 Agent 此时的上下文状态：
  - 系统提示
  - 整个任务的规划过程（已有几十个 turn）
  
若主 Agent 直接读取所有产物：
  + corrections_parsed.json（结构化更正项）
  + state_law_findings.json（法规查询结果）
  + city_research_findings.json（城市法规）
  + sheet_observations.json（图纸观察）
  → 可能超出上下文限制，且推理质量下降

解决：启动专用合并子 Agent（干净上下文）
```

### 合并子 Agent 的工作

```
Phase 4 合并子 Agent（独立上下文）
    │
    ├─ 读取 corrections_parsed.json
    ├─ 读取 state_law_findings.json
    ├─ 读取 city_research_findings.json
    ├─ 读取 sheet_observations.json
    │
    ├─ 逐项处理（14 条更正项）：
    │   ├─ 找到该条的法规依据（来自 state_law + city）
    │   ├─ 找到该条引用图纸的观察结果（来自 sheet_obs）
    │   ├─ 综合分类：AUTO_FIXABLE / NEEDS_INPUT / NEEDS_PROFESSIONAL
    │   └─ 生成承包商问题（带上下文和选项）
    │
    ├─ 输出 corrections_categorized.json
    └─ 输出 contractor_questions.json
    
    → 合并子 Agent 完成，上下文释放
```

**合并子 Agent 的过滤规则（城市审查 Phase 4）：**

```markdown
# 来自 adu-plan-review/SKILL.md

过滤规则（必须严格遵守）：

| 条件 | 行为 |
|------|------|
| 发现被州法 AND/OR 城市法确认 | ✅ 包含（附引用）|
| 发现被确认，但视觉置信度 LOW | ✅ 包含（标注 [VERIFY]）|
| 发现未被任何法规确认 | ❌ 丢弃 |
| 涉及工程充分性判断 | ✅ 包含（标注 [REVIEWER: ...]）|
| 需要主观判断的发现 | ❌ 丢弃（Gov. Code §66314(b)(1) 禁止）|
```

---

## 8. 时序图：完整并行执行流程

以**更正分析 Phase 3**为例的详细时序：

```
时间      主 Agent         子 Agent 3A       子 Agent 3B       子 Agent 3C × n
                          （州法规）         （城市发现）        （图纸观察）

 t=0      Phase 2 完成
          读取 sheet-manifest.json
          
 t=30     启动 3A, 3B, 3C ─────────────────────────────────────────────────→ 启动
           ↓                  ↓                  ↓                  ↓
           等待              查询 california-adu  WebSearch 城市     读取 Sheet A3
                            技能中的 28 个文件    ADU 页面           PNG 文件（视觉）
                            
 t=60     等待中...          查询法规条款...      WebSearch 完成     Sheet A3 分析完成
                                                写 city_discovery  写 sheet_obs_A3.json
                                                                   
                                                                   启动下一批图纸子 Agent
                                                                   读取 Sheet C2 PNG...

 t=90     等待中...          州法查询完成         
                            写 state_law_findings.json
                            
 t=120    3A, 3B, 3C 全部完成
          验证产物文件存在
          （不读取内容）
          
          启动 Phase 3.5：
          城市 WebFetch 子 Agent
           ↓
          读取 city_discovery 中的 URL
          WebFetch 提取法规全文
          写 city_research_findings.json

 t=210    Phase 3.5 完成
          
          启动 Phase 4：合并子 Agent
           ↓
          读取全部产物文件
          逐项分类 + 生成问题
          写 corrections_categorized.json
          写 contractor_questions.json

 t=330    Phase 4 完成
          Agent 流程结束
          写入 Supabase
```

**总计：约 5.5 分钟（无回退）**

对比串行方案：约 21 分钟

---

## 9. 并行参数与预算控制

### 预算如何分配到子 Agent

系统在 Cloud Run 层设置总预算（`maxTurns`, `maxBudgetUsd`），Agent SDK 统一跟踪：

```typescript
// 在 sandbox.ts 中
const agentOptions = {
  model: 'claude-opus-4-6',
  maxTurns: 500,       // 总 turn 数上限（主 Agent + 所有子 Agent）
  maxBudgetUsd: 50,    // 总费用上限
  permissionMode: 'bypassPermissions',
  preset: 'claude_code',
}
```

**子 Agent 消耗主 Agent 的预算：**

```
总预算：500 turns, $50

主 Agent：约 50-80 turns
子 Agent（Phase 3A）：约 20-30 turns
子 Agent（Phase 3B）：约 10-20 turns  
子 Agent（Phase 3C）× n：约 5-10 turns × n
Phase 4 子 Agent：约 30-50 turns

典型用量：~150-250 turns，$3-8
远低于上限，有充足余量应对复杂案例
```

### 滚动窗口大小选择依据

| 窗口大小 | 并发度 | 风险 | 适用场景 |
|---------|-------|------|---------|
| 1（串行）| 低 | 速率限制风险最低 | 调试、简单任务 |
| 3（当前）| 中 | 速率限制风险低 | **生产默认** |
| 5 | 高 | 可能触发速率限制 | 高吞吐需求 |

当前选择 **3** 的理由：
1. Anthropic API 并发请求有速率限制（requests per minute）
2. 3 个并发在实测中不触发 429 错误
3. 已提供 2-3× 加速，收益边际递减

---

## 10. 与外层并行的区别

系统中存在两个层面的并行，不要混淆：

| 层面 | 机制 | 发生位置 | 用途 |
|------|------|---------|------|
| **Agent 内部并行** | Task 工具（本文档重点）| Vercel Sandbox 内 | 多专业研究、图纸分析 |
| **外层任务并行** | Cloud Run 多实例 | GCP 基础设施层 | 多个用户项目同时运行 |

**外层并行（不同用户的项目）：**

```
用户 A 的项目 → Cloud Run 实例 1 → Sandbox A
用户 B 的项目 → Cloud Run 实例 2 → Sandbox B  （完全隔离）
用户 C 的项目 → Cloud Run 实例 3 → Sandbox C
```

每个项目有独立的 Sandbox，完全隔离，无需协调。

**Agent 内部并行（单个项目内）：**

```
Sandbox（单个项目）
  └─ 主 Agent
       ├─ Task → 子 Agent A（共享 Sandbox 文件系统）
       ├─ Task → 子 Agent B（共享 Sandbox 文件系统）
       └─ Task → 子 Agent C（共享 Sandbox 文件系统）
```

文件系统是子 Agent 之间唯一的共享状态，无需锁机制（各子 Agent 写不同文件）。

---

## 总结

CrossBeam 的多 Agent 并行执行通过以下机制实现：

1. **Task 工具** — Agent SDK 提供的子 Agent spawn 机制，每个子 Agent 有独立上下文
2. **文件系统中间层** — 子 Agent 将结果写入文件，避免大数据通过对话返回
3. **滚动并发窗口（N=3）** — 控制并发度，避免 API 速率限制
4. **主 Agent 只做协调** — 不读取大型产物，只验证文件是否存在
5. **专用合并子 Agent** — Phase 4 用干净上下文聚合所有研究结果
6. **Sandbox 共享文件系统** — 唯一的进程间通信方式，简单可靠

这套架构使得总执行时间从串行的 35+ 分钟降至 10-15 分钟，同时保持了上下文的清洁性和推理质量。

---

*文档版本：v1.0 | 2026-05-11*  
*对应源码：`server/skills/adu-corrections-flow/SKILL.md`, `server/skills/adu-plan-review/SKILL.md`, `server/src/services/sandbox.ts`*
