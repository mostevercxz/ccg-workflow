---
description: '多模型协作规划（回合制收敛）- 上下文检索 + A/B 交叉审查 + 迭代落盘，最终生成可执行计划'
---

# Plan - 多模型协作规划（回合制收敛）

$ARGUMENTS

---

## 核心协议

- **语言协议**：与工具/模型交互用**英语**，与用户交互用**中文**
- **强制并行**：A/B 双模型调用必须使用 `run_in_background: true`（包含单模型调用，避免阻塞主线程）
- **代码主权**：外部模型对文件系统**零写入权限**，所有修改由 Claude 执行
- **止损机制**：当前阶段输出通过验证前，不进入下一阶段
- **仅规划**：本命令允许读取上下文与写入 `.claude/plan/*` 计划文件，但**禁止修改产品代码**
- **回合制收敛**：规划按 Round 迭代（A 设计 → 用户确认 → B 审查 → A 修订）直到收敛
- **强制任务拆分**：每轮与最终计划都必须输出“子任务清单 + 文件边界 + 验收标准”，禁止仅给高层描述
- **原始输入保护**：用户最原始输入必须逐字保留到 `meta.json`，禁止改写和覆盖

---

## 计划存储规范（必须遵循）

### 目录结构

```text
.claude/plan/tasks/<task_id>/
  meta.json
  rounds/
    round-1/
      summary.md
      questions.md
      issues.md
      changelog.md
      artifacts/
        architecture/
        api/
        db/
        diagrams/
    round-2/
      ...
  final/
    final-plan.md
    decisions.md
    unresolved.md
```

### task_id 规则（自动生成）

- **禁止要求用户手填 `<feature>`**
- 规则：`<slug>-<hash6>`
  - `slug`：从需求标题/首行自动提取，可读短名（建议 6~24 字符）
  - `hash6`：基于原始输入全文计算哈希后取前 6 位
- 示例：`user-auth-3fa9c2`、`order-settlement-c81d4e`

### meta.json（最小字段，必须包含）

```json
{
  "task_id": "<slug-hash6>",
  "title": "<自动提取标题>",
  "created_at": "<ISO8601>",
  "updated_at": "<ISO8601>",
  "last_round": 1,
  "status": "planning",
  "original_input": {
    "format": "markdown",
    "content": "<用户最原始输入全文，逐字保留>",
    "sha256": "<原文哈希>",
    "length_chars": 12345
  },
  "input_attachments": []
}
```

**硬性要求**：
- `original_input.content` 必须保留原文全文（verbatim）
- 后续轮次禁止修改 `original_input`，仅更新 `updated_at`、`last_round`、`status`

---

## 多模型调用规范

**工作目录**：
- `{{WORKDIR}}`：替换为目标工作目录的**绝对路径**
- 如果用户通过 `/add-dir` 添加了多个工作区，先用 Glob/Grep 确定任务相关的工作区
- 如果无法确定，用 `AskUserQuestion` 询问用户选择目标工作区
- 默认使用当前工作目录

**调用语法**（并行用 `run_in_background: true`）：

```
Bash({
  command: "~/.claude/bin/codeagent-wrapper {{LITE_MODE_FLAG}}--backend <claude|codex|gemini> {{GEMINI_MODEL_FLAG}}- \"{{WORKDIR}}\" <<'EOF'
ROLE_FILE: <角色提示词路径>
<TASK>
需求：<增强后的需求>
上下文：<检索到的项目上下文>
Round: <N>
Mode: <author|reviewer>
输出格式：严格按指定模板输出
</TASK>
OUTPUT: Planning artifacts ONLY. DO NOT modify any files.
EOF",
  run_in_background: true,
  timeout: 3600000,
  description: "简短描述"
})
```

**模型参数说明**：
- `{{GEMINI_MODEL_FLAG}}`：当使用 `--backend gemini` 时，替换为 `--gemini-model gemini-3.1-pro-preview `（注意末尾空格）；使用 `claude/codex` 时替换为空字符串

**角色提示词**：

| 阶段 | Claude | Codex | Gemini |
|------|--------|-------|--------|
| 分析 | `~/.claude/.ccg/prompts/claude/analyzer.md` | `~/.claude/.ccg/prompts/codex/analyzer.md` | `~/.claude/.ccg/prompts/gemini/analyzer.md` |
| 规划 | `~/.claude/.ccg/prompts/claude/architect.md` | `~/.claude/.ccg/prompts/codex/architect.md` | `~/.claude/.ccg/prompts/gemini/architect.md` |

**会话复用**：每次调用返回 `SESSION_ID: xxx`（通常由 wrapper 输出），**必须保存**供后续轮次与 `/ccg:execute` 使用；最终计划需按实际使用模型记录对应 SESSION（如 `CLAUDE_SESSION` / `CODEX_SESSION` / `GEMINI_SESSION`）。

**等待后台任务**（最大超时 600000ms = 10 分钟）：

```
TaskOutput({ task_id: "<task_id>", block: true, timeout: 600000 })
```

**重要**：
- 必须指定 `timeout: 600000`，否则默认只有 30 秒会导致提前超时
- 若 10 分钟后仍未完成，继续用 `TaskOutput` 轮询，**绝对不要 Kill 进程**
- 若因等待时间过长跳过了等待，**必须调用 `AskUserQuestion` 询问用户选择继续等待还是 Kill Task**

---

## 执行工作流

**规划任务**：$ARGUMENTS

### 🧱 Phase 0：初始化任务空间（必须先做）

`[模式：准备]`

1. 执行 Prompt 增强（按 `/ccg:enhance` 逻辑）：补全目标、约束、边界、验收标准
2. 基于原始输入自动生成 `task_id=<slug>-<hash6>`
3. 创建目录：`.claude/plan/tasks/<task_id>/`
4. 写入 `meta.json`（必须包含 `original_input.content` 全文）
5. 初始化 `rounds/round-1/` 目录与基础文件（可为空模板）

### 🔍 Phase 1：上下文全量检索

`[模式：研究]`

**调用 `{{MCP_SEARCH_TOOL}}` 工具**：

```
{{MCP_SEARCH_TOOL}}({
  query: "<基于增强后需求构建的语义查询>",
  project_root_path: "{{WORKDIR}}"
})
```

- 使用自然语言构建语义查询（Where/What/How）
- **禁止基于假设回答**
- 若 MCP 不可用：回退到 Glob + Grep 进行文件发现与关键符号定位
- 必须获取关键类/函数/变量的完整定义；不足则递归检索

### 🔁 Phase 2：回合制协作规划（A/B 循环）

`[模式：分析+规划]`

#### 2.1 角色分配（默认规则）

- **后端主导任务**：A=Claude，B=Codex
- **前端主导任务**：A=Gemini，B=Codex
- **全栈任务**：按模块拆分后分别应用同样规则（前端模块 A=Gemini，后端模块 A=Claude，统一 B=Codex）

#### 2.2 每一轮固定步骤（Round N）

**回合与窗口规则（防无限循环）**：
- 每个 Round 只允许：
  - **1 次确认窗口**（处理 `questions.md`）
  - **1 次裁决窗口**（处理 `issues.md`）
- **一次窗口可批量处理多个问题**（例如 `Q1..Qn` / `I1..Im`）
- A 在本轮末输出 `vN.1` 并落盘后，**本轮必须立即结束**
- 若 `vN.1` 产生新待确认问题或新分歧，**强制进入 Round N+1**，禁止留在本轮继续开新窗口

1. **A 产出方案草案**（author）：
   - 输出 `summary.md`（方案正文）
   - 输出 `questions.md`（需要用户确认的问题，必须编号）
2. **用户确认问题（确认窗口）**：
   - 一次性批量回复 `Q1..Qn`
   - 将用户答复写入 `round-N/changelog.md` 或 `final/decisions.md`（若是全局决策）
3. **B 交叉审查**（reviewer）：
   - 输出 `issues.md`（问题清单，含严重级别：blocker/high/medium/low）
4. **用户裁决 issues（裁决窗口）**：
   - 一次性批量裁决 `I1..Im`（accepted/rejected/deferred）
   - 将裁决结果写入 `round-N/changelog.md`
5. **A 基于 B + 用户反馈修订**：
   - 更新 `summary.md`
   - 在 `changelog.md` 明确“已修复/暂不采纳/待确认”
   - 输出本轮收敛稿 `vN.1`
6. **落盘产物**：
   - 允许在 `artifacts/` 下生成架构图、接口草案、目录结构文档等多文件产物

#### 2.2.1 结构化输出要求（强制）

每轮 `summary.md` 必须包含以下结构（缺一不可）：

```markdown
## 子任务清单（Task Breakdown）

### Task 1: <名称>
- **类型**: 前端/后端/全栈
- **文件范围**: <精确文件路径列表>
- **依赖**: 无 / Task N
- **实施步骤**:
  1. <具体步骤>
  2. <具体步骤>
- **验收标准**:
  - [ ] <可验证条件 1>
  - [ ] <可验证条件 2>

### Task 2: <名称>
...

## 文件冲突检查
✅ 无冲突 / ⚠️ 已通过依赖关系解决

## 并行分组（可选）
- Layer 1 (并行): Task 1, Task 2
- Layer 2 (依赖 Layer 1): Task 3
```

硬性约束：
- 子任务必须可执行、可验收，禁止“空泛描述型任务”
- 文件范围必须明确到文件路径（必要时到函数/区块）
- 若文件范围重叠，必须显式声明依赖顺序或合并任务
- 每个子任务必须有至少 1 条可验证验收标准

#### 2.3 收敛判定（必须显式检查）

满足以下条件才可结束规划：

- `blocker/high` 问题为 0
- `questions.md` 中待确认问题为 0
- 关键约束均已决策（记录在 `final/decisions.md`）
- `summary.md` 与 `final-plan.md` 均包含完整“子任务清单 + 文件边界 + 验收标准”结构

若未收敛：
- `last_round += 1`
- 创建下一轮目录 `rounds/round-(N+1)/`
- 继续 Phase 2.2

若达到最大轮次仍未收敛：
- 必须调用 `AskUserQuestion` 请求用户决策（继续迭代 / 降级范围 / 中止）

### 📦 Phase 3：收敛交付（仅交付计划，不执行）

`[模式：交付]`

收敛后必须执行：

1. 生成 `final/final-plan.md`（最终可执行计划）
   - **必须包含**：子任务清单、文件边界、实施步骤、验收标准、依赖关系、风险与缓解
2. 生成 `final/decisions.md`（关键决策）
3. 生成 `final/unresolved.md`（遗留风险，若为空也要写“无”）
4. 更新 `meta.json`：`status=approved`、`updated_at`、`last_round`
5. 向用户展示最终计划，并提示执行命令：

```bash
/ccg:execute .claude/plan/tasks/<task_id>/final/final-plan.md
```

**`/ccg:plan` 的职责到此结束，必须立即停止（Stop here. No more tool calls.）**

---

## 绝对禁止

- ❌ 问用户 "Y/N" 后自动执行（执行是 `/ccg:execute` 的职责）
- ❌ 对产品代码进行任何写操作
- ❌ 自动调用 `/ccg:execute` 或任何实施动作
- ❌ 在用户未要求进入下一轮时，继续触发模型调用

---

## 计划修改流程（已存在任务）

如果用户要求继续优化已收敛计划：

1. 读取 `.claude/plan/tasks/<task_id>/meta.json`
2. 创建新轮次目录 `rounds/round-(last_round+1)/`
3. 继续执行 Phase 2（回合制）
4. 重新写入 `final/*` 与 `meta.json`

---

## 关键规则

1. **仅规划不实施** – 本命令不执行任何代码变更
2. **回合制收敛** – A/B + 用户确认必须形成闭环，不能只跑单轮
3. **强制拆分输出** – 每轮与最终计划必须包含“子任务清单 + 文件边界 + 验收标准”
4. **信任规则** – 审查结论以 Codex 为主；前端规划可优先采纳 Gemini，后端规划由 Claude 主导整合
5. **外部模型零写入** – 文件修改与落盘由 Claude 执行
6. **SESSION_ID 交接** – final 计划必须包含实际使用模型的会话标识（如 `CLAUDE_SESSION` / `CODEX_SESSION` / `GEMINI_SESSION`，供 `/ccg:execute resume <SESSION_ID>` 使用）
