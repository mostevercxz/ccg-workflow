---
description: '多模型协作执行 - 基于 tasks/<task_id>/final/final-plan.md 实施代码变更并完成双模型审计交付'
---

# Execute - 多模型协作执行

$ARGUMENTS

---

## 核心协议

- **语言协议**：与工具/模型交互用**英语**，与用户交互用**中文**
- **代码主权**：外部模型对文件系统**零写入权限**，所有修改由 Claude 执行
- **脏原型重构**：将 Codex/Gemini 的 Unified Diff 视为“脏原型”，必须重构为生产级代码
- **止损机制**：当前阶段输出通过验证前，不进入下一阶段
- **计划来源约束**：优先执行 `/ccg:plan` 收敛后的最终计划：`.claude/plan/tasks/<task_id>/final/final-plan.md`

---

## 多模型调用规范

**工作目录**：
- `{{WORKDIR}}`：替换为目标工作目录的**绝对路径**
- 如果用户通过 `/add-dir` 添加了多个工作区，先用 Glob/Grep 确定任务相关的工作区
- 如果无法确定，用 `AskUserQuestion` 询问用户选择目标工作区
- 默认使用当前工作目录

**调用语法**（并行用 `run_in_background: true`）：

```
# 复用会话调用（推荐）- 原型生成（Implementation Prototype）
Bash({
  command: "~/.claude/bin/codeagent-wrapper {{LITE_MODE_FLAG}}--backend <codex|gemini> {{GEMINI_MODEL_FLAG}}resume <SESSION_ID> - \"{{WORKDIR}}\" <<'EOF'
ROLE_FILE: <角色提示词路径>
<TASK>
需求：<任务描述>
上下文：<final-plan 内容 + 目标文件>
</TASK>
OUTPUT: Unified Diff Patch ONLY. Strictly prohibit any actual modifications.
EOF",
  run_in_background: true,
  timeout: 3600000,
  description: "简短描述"
})

# 新会话调用 - 原型生成（Implementation Prototype）
Bash({
  command: "~/.claude/bin/codeagent-wrapper {{LITE_MODE_FLAG}}--backend <codex|gemini> {{GEMINI_MODEL_FLAG}}- \"{{WORKDIR}}\" <<'EOF'
ROLE_FILE: <角色提示词路径>
<TASK>
需求：<任务描述>
上下文：<final-plan 内容 + 目标文件>
</TASK>
OUTPUT: Unified Diff Patch ONLY. Strictly prohibit any actual modifications.
EOF",
  run_in_background: true,
  timeout: 3600000,
  description: "简短描述"
})
```

**审计调用语法**（Code Review / Audit）：

```
Bash({
  command: "~/.claude/bin/codeagent-wrapper {{LITE_MODE_FLAG}}--backend <codex|gemini> {{GEMINI_MODEL_FLAG}}resume <SESSION_ID> - \"{{WORKDIR}}\" <<'EOF'
ROLE_FILE: <角色提示词路径>
<TASK>
Scope: Audit the final code changes.
Inputs:
- The applied patch (git diff / final unified diff)
- The touched files (relevant excerpts if needed)
Constraints:
- Do NOT modify any files.
- Do NOT output tool commands that assume filesystem access.
</TASK>
OUTPUT:
1) A prioritized list of issues (severity, file, rationale)
2) Concrete fixes; if code changes are needed, include a Unified Diff Patch in a fenced code block.
EOF",
  run_in_background: true,
  timeout: 3600000,
  description: "简短描述"
})
```

**模型参数说明**：
- `{{GEMINI_MODEL_FLAG}}`：当使用 `--backend gemini` 时，替换为 `--gemini-model gemini-3.1-pro-preview `（注意末尾空格）；使用 codex 时替换为空字符串

**角色提示词**：

| 阶段 | Codex | Gemini |
|------|-------|--------|
| 实施 | `~/.claude/.ccg/prompts/codex/architect.md` | `~/.claude/.ccg/prompts/gemini/frontend.md` |
| 审查 | `~/.claude/.ccg/prompts/codex/reviewer.md` | `~/.claude/.ccg/prompts/gemini/reviewer.md` |

**会话复用**：如果 `final-plan.md` 提供了 SESSION_ID，使用 `resume <SESSION_ID>` 复用上下文。

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

**执行任务**：$ARGUMENTS

### 📖 Phase 0：读取计划与任务元数据

`[模式：准备]`

1. **识别输入类型**：
   - 推荐：`/ccg:execute .claude/plan/tasks/<task_id>/final/final-plan.md`
   - 兼容：旧格式 `.claude/plan/<feature>.md`（建议迁移）
   - 直接任务描述（不推荐）

2. **读取并校验计划上下文**：
   - 读取 `final-plan.md`
   - 若存在，读取同目录上级的 `meta.json`
   - 校验 `meta.status` 是否为 `approved`（若不是，先 AskUserQuestion 确认是否继续）

3. **提取执行信息**：
   - 任务类型、实施步骤、关键文件、风险缓解
   - `CODEX_SESSION`、`GEMINI_SESSION`

4. **执行前确认**：
   - 若输入为直接任务描述，或计划缺失关键字段（步骤/关键文件/SESSION_ID），必须先向用户确认补全

5. **任务类型判断**：

   | 任务类型 | 判断依据 | 路由 |
   |----------|----------|------|
   | **前端** | 页面、组件、UI、样式、布局 | Gemini |
   | **后端** | API、接口、数据库、逻辑、算法 | Codex |
   | **全栈** | 同时包含前后端 | Codex ∥ Gemini 并行 |

---

### 🔍 Phase 1：上下文快速检索

`[模式：检索]`

**⚠️ 必须使用 MCP 工具快速检索上下文，禁止手动逐个读取文件**

根据计划中的“关键文件”列表，调用 `{{MCP_SEARCH_TOOL}}` 检索相关代码：

```
{{MCP_SEARCH_TOOL}}({
  query: "<基于 final-plan 内容构建的语义查询，包含关键文件、模块、函数名>",
  project_root_path: "{{WORKDIR}}"
})
```

**检索策略**：
- 从计划的“关键文件”提取目标路径
- 构建语义查询覆盖：入口文件、依赖模块、相关类型定义
- 若检索结果不足，可追加 1-2 次递归检索
- **禁止**使用 Bash + find/ls 手动探索项目结构

---

### 🎨 Phase 2：原型获取

`[模式：原型]`

#### Route A: 前端/UI/样式 → Gemini

1. 调用 Gemini（`~/.claude/.ccg/prompts/gemini/frontend.md`）
2. 输入：final-plan + 检索上下文 + 目标文件
3. OUTPUT: `Unified Diff Patch ONLY. Strictly prohibit any actual modifications.`
4. 若 final-plan 包含 `GEMINI_SESSION`：优先 `resume <GEMINI_SESSION>`

#### Route B: 后端/逻辑/算法 → Codex

1. 调用 Codex（`~/.claude/.ccg/prompts/codex/architect.md`）
2. 输入：final-plan + 检索上下文 + 目标文件
3. OUTPUT: `Unified Diff Patch ONLY. Strictly prohibit any actual modifications.`
4. 若 final-plan 包含 `CODEX_SESSION`：优先 `resume <CODEX_SESSION>`

#### Route C: 全栈 → 并行调用

1. 并行调用 Codex + Gemini（`run_in_background: true`）
2. 使用 `TaskOutput` 等待两个模型完整结果
3. 优先复用各自 SESSION_ID

---

### ⚡ Phase 3：编码实施

`[模式：实施]`

1. **读取 Diff**：解析模型返回的 Unified Diff
2. **思维沙箱**：模拟应用、检查一致性与副作用
3. **重构清理**：将“脏原型”重构为可维护、可发布代码
4. **最小作用域**：仅修改需求范围，强制检查副作用
5. **应用变更**：使用 Edit/Write 执行实际修改
6. **自检验证**：运行 lint / typecheck / tests（优先最小相关范围）
   - 若失败：优先修复后再进入下一阶段

---

### ✅ Phase 4：审计与交付

`[模式：审计]`

#### 4.1 自动审计（强制）

变更生效后，必须并行调用 Codex + Gemini 进行 Code Review：
- Codex：安全性、性能、错误处理、逻辑正确性
- Gemini：可访问性、设计一致性、用户体验

使用 `TaskOutput` 等待完整审查结果，优先复用 Phase 2 会话。

#### 4.2 整合修复

1. 综合 Codex + Gemini 审查意见
2. 按信任规则权衡：后端以 Codex 为准，前端以 Gemini 为准
3. 执行必要修复
4. 按需重复 4.1，直到风险可接受

#### 4.3 交付确认 + 执行报告落盘（新增）

审计通过后，必须在任务目录写入：

` .claude/plan/tasks/<task_id>/final/execution-report.json `

最小字段建议：

```json
{
  "task_id": "<task_id>",
  "executed_at": "<ISO8601>",
  "plan_path": ".claude/plan/tasks/<task_id>/final/final-plan.md",
  "status": "done",
  "validation": {
    "lint": "pass|fail|na",
    "typecheck": "pass|fail|na",
    "tests": "pass|fail|na"
  },
  "audit": {
    "codex": "pass|issues",
    "gemini": "pass|issues"
  },
  "changed_files": []
}
```

并向用户报告执行摘要：

```markdown
## ✅ 执行完成

### 变更摘要
| 文件 | 操作 | 说明 |
|------|------|------|
| path/to/file.ts | 修改 | 描述 |

### 验证结果
- lint: <pass/fail/na>
- typecheck: <pass/fail/na>
- tests: <pass/fail/na>

### 审计结果
- Codex：<通过/发现 N 个问题>
- Gemini：<通过/发现 N 个问题>
```

---

## 关键规则

1. **代码主权** – 所有文件修改由 Claude 执行，外部模型零写入权限
2. **脏原型重构** – 模型输出仅为草稿，必须重构
3. **信任规则** – 后端以 Codex 为准，前端以 Gemini 为准
4. **最小变更** – 仅修改必要代码，不引入副作用
5. **强制审计** – 变更后必须进行多模型 Code Review
6. **优先 final-plan** – 执行以 `.claude/plan/tasks/<task_id>/final/final-plan.md` 为准

---

## 使用方法

```bash
# 推荐：执行收敛后的最终计划
/ccg:execute .claude/plan/tasks/<task_id>/final/final-plan.md

# 兼容旧格式（建议迁移）
/ccg:execute .claude/plan/<功能名>.md
```

---

## 与 /ccg:plan 的关系

1. `/ccg:plan` 在 `.claude/plan/tasks/<task_id>/` 下完成回合制收敛
2. 生成 `final/final-plan.md` + `meta.json(status=approved)`
3. `/ccg:execute` 读取 final-plan，复用 SESSION_ID，执行实施
