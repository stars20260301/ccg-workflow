---
description: 'CCG 多模型协作工作流 - Claude 主导，Codex+Gemini 协作，自动适配任务规模'
---

# Workflow - CCG 多模型协作工作流

$ARGUMENTS

---

## 协作模型

Claude 是主导者（决策、编排、写码），Codex 是后端军师（分析、建议、审查），Gemini 是前端军师（分析、建议、审查）。

原则：
- Claude 主导每个环节，但关键节点必须咨询军师
- 军师意见不可忽略，采纳或不采纳必须说明理由
- 后端分歧听 Codex，前端分歧听 Gemini
- 军师对文件系统零写入权限，所有代码由 Claude 或 Builder 执行

---

## 核心协议

- 语言协议：与工具/模型交互用英语，与用户交互用中文
- 代码主权：外部模型零写入权限，所有修改由 Claude 执行
- 咨询义务：方案选型、计划定稿、最终审查必须咨询双军师；不采纳须给理由
- 脏原型重构：Codex/Gemini 的 Unified Diff 是参考草稿，Claude 重构为生产级代码
- 止损机制：当前阶段输出通过验证前，不进入下一阶段

---

## 多模型调用规范

工作目录：
- `{{WORKDIR}}`：替换为目标工作目录的绝对路径
- 如果用户通过 `/add-dir` 添加了多个工作区，先用 Glob/Grep 确定任务相关的工作区
- 如果无法确定，用 `AskUserQuestion` 询问用户选择目标工作区
- 默认使用当前工作目录

调用语法（并行用 `run_in_background: true`）：

```
# 新会话
Bash({
  command: "~/.claude/bin/codeagent-wrapper {{LITE_MODE_FLAG}}--backend <codex|gemini> {{GEMINI_MODEL_FLAG}}- \"{{WORKDIR}}\" <<'EOF'
ROLE_FILE: <角色提示词路径>
<TASK>
需求：<任务描述>
上下文：<项目上下文>
</TASK>
OUTPUT: <期望输出格式>
EOF",
  run_in_background: true,
  timeout: 3600000,
  description: "简短描述"
})

# 复用会话
Bash({
  command: "~/.claude/bin/codeagent-wrapper {{LITE_MODE_FLAG}}--backend <codex|gemini> {{GEMINI_MODEL_FLAG}}resume <SESSION_ID> - \"{{WORKDIR}}\" <<'EOF'
ROLE_FILE: <角色提示词路径>
<TASK>
需求：<任务描述>
上下文：<项目上下文>
</TASK>
OUTPUT: <期望输出格式>
EOF",
  run_in_background: true,
  timeout: 3600000,
  description: "简短描述"
})
```

模型参数：`{{GEMINI_MODEL_FLAG}}` 当使用 `--backend gemini` 时替换为 `--gemini-model gemini-3-pro-preview `（注意末尾空格），使用 codex 时替换为空字符串。

角色提示词：

| 阶段 | Codex | Gemini |
|------|-------|--------|
| 分析 | `~/.claude/.ccg/prompts/codex/analyzer.md` | `~/.claude/.ccg/prompts/gemini/analyzer.md` |
| 规划 | `~/.claude/.ccg/prompts/codex/architect.md` | `~/.claude/.ccg/prompts/gemini/architect.md` |
| 实施 | `~/.claude/.ccg/prompts/codex/architect.md` | `~/.claude/.ccg/prompts/gemini/frontend.md` |
| 审查 | `~/.claude/.ccg/prompts/codex/reviewer.md` | `~/.claude/.ccg/prompts/gemini/reviewer.md` |

SESSION_ID：每次调用返回 `SESSION_ID: xxx`，必须保存，后续阶段用 `resume <SESSION_ID>` 复用上下文。

等待后台任务（最大超时 600000ms = 10 分钟）：

```
TaskOutput({ task_id: "<task_id>", block: true, timeout: 600000 })
```

必须指定 `timeout: 600000`，否则默认只有 30 秒会导致提前超时。若 10 分钟后仍未完成，继续用 `TaskOutput` 轮询，绝对不要 Kill 进程。若因等待时间过长跳过了等待，必须调用 `AskUserQuestion` 询问用户选择继续等待还是 Kill Task。

---

## 执行工作流

开发任务：$ARGUMENTS

---

### Phase 0：需求理解 + 规模判断

0.1 识别输入类型

- 计划文件路径（`.claude/plan/*.md`）→ 读取计划，跳到 Phase 3
- 自然语言需求 → 继续 0.2

0.2 Prompt 增强（必须执行）

分析 $ARGUMENTS 的意图、缺失信息、隐含假设，补全为结构化需求（明确目标、技术约束、范围边界、验收标准）。用增强结果替代原始 $ARGUMENTS 用于后续所有阶段。

0.3 上下文检索

调用 `{{MCP_SEARCH_TOOL}}` 检索项目上下文：

```
{{MCP_SEARCH_TOOL}}({
  query: "<基于增强后需求构建的语义查询>",
  project_root_path: "{{WORKDIR}}"
})
```

- 使用自然语言构建语义查询（Where/What/How）
- 若 MCP 不可用：回退到 Glob + Grep
- 必须获取相关类、函数、变量的完整定义与签名
- 若上下文不足，触发递归检索

0.4 需求对齐

若需求仍有模糊空间，必须用 `AskUserQuestion` 向用户输出引导性问题列表，直至需求边界清晰。

0.5 规模判断

| 规模 | 判断依据 | 实施方式 |
|------|----------|----------|
| S | 单文件改动 / 简单 bug 修复 / 配置调整 | Claude 直接干（Phase 3S） |
| M/L | 2+ 文件 / 跨模块 | 团队模式：spawn Builder（Phase 3T） |

Lead（Opus）的上下文留给决策和协调，写码交给 Builder。S 级太小不值得 spawn，Claude 直接干。M/L 级一律走团队模式。

---

### Phase 1：分析（M/L 级）

1.1 Claude 先形成初步判断

基于 Phase 0 的上下文，Claude 先独立思考初步技术方案、预判关键风险、识别需要军师意见的点。

1.2 咨询军师（并行）

在同一条消息中发起两个 Bash 调用：

1. Codex 后端分析（`run_in_background: true`）：
   - ROLE_FILE: `~/.claude/.ccg/prompts/codex/analyzer.md`
   - 输入：增强后的需求 + 项目上下文
   - OUTPUT: 技术可行性、架构影响、性能考量、风险评估

2. Gemini 前端分析（`run_in_background: true`）：
   - ROLE_FILE: `~/.claude/.ccg/prompts/gemini/analyzer.md`
   - 输入：增强后的需求 + 项目上下文
   - OUTPUT: UI/UX 影响、组件拆分、交互设计、可访问性

用 `TaskOutput` 等待两个模型的完整结果。保存 `CODEX_SESSION` 和 `GEMINI_SESSION`。

1.3 Claude 交叉验证

整合 Claude 自身判断 + 双军师意见：

1. 一致点（强信号）→ 直接采纳
2. 分歧点 → 按信任规则裁决：后端听 Codex，前端听 Gemini
3. 互补优势 → 吸收各方独到见解
4. 不采纳的意见 → 记录理由

产出：约束集（硬约束 + 软约束 + 依赖关系 + 风险）

1.4 歧义消解

从约束集和军师意见中提取所有开放问题（未确定的技术选型、模糊的边界条件、隐含假设等），用 `AskUserQuestion` 系统性呈现：
- 分组相关问题
- 为每个问题提供上下文
- 在适用时建议默认值

将用户回答转化为额外的硬约束或软约束，补入约束集。无开放问题后方可进入 Phase 2。

---

### Phase 2：规划（M/L 级）

2.1 咨询军师出计划草案（并行，推荐）

resume SESSION_ID，让军师在已有上下文基础上出草案：

1. Codex 计划草案（`resume CODEX_SESSION`）：
   - ROLE_FILE: `~/.claude/.ccg/prompts/codex/architect.md`
   - OUTPUT: Step-by-step plan（重点：数据流/边界条件/错误处理/测试策略）

2. Gemini 计划草案（`resume GEMINI_SESSION`）：
   - ROLE_FILE: `~/.claude/.ccg/prompts/gemini/architect.md`
   - OUTPUT: Step-by-step plan（重点：信息架构/交互/可访问性/视觉一致性）

2.2 Claude 综合定稿

Claude 参考双方草案，自己写最终计划。计划必须包含：

- 任务类型（前端/后端/全栈）
- 约束集（硬约束 + 软约束）
- 技术方案（说明采纳/不采纳军师意见的理由）
- 实施步骤（每步有预期产物）
- 关键文件表（文件路径:行号 + 操作 + 说明）
- 风险与缓解
- SESSION_ID（CODEX_SESSION + GEMINI_SESSION）

2.3 并行分组（M/L 级均需）

拆分为并行分组：

- 按文件范围隔离拆分子任务（文件不可重叠）
- 若无法避免重叠 → 设为依赖关系
- 按依赖关系分 Layer：同 Layer 可并行，跨 Layer 串行
- 为每个子任务标注推荐 Builder 模型（见 3T.1）

2.4 用户确认

展示完整计划，等待用户确认。用户未确认前，禁止进入 Phase 3。

---

### Phase 3S：S 级快速实施

Claude 独立完成：直接编码 → 自检验证（lint/typecheck/tests）→ 报告变更摘要 → 进入 Phase 4。

---

### Phase 3T：团队模式实施（Agent Teams）

3T.0 前置检查

- 检测 Agent Teams 是否可用（`CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`）
- 若不可用 → 提示用户启用，或降级为 S 级模式由 Claude 线性执行

3T.1 Builder 模型选择

Lead 始终是当前模型（Opus），Builder 按子任务复杂度选模型：

| 复杂度 | 模型 | 判断依据 |
|--------|------|----------|
| 简单 | Haiku | 单文件、配置、样板代码、CRUD |
| 中等 | Sonnet | 多文件联动、业务逻辑、组件实现 |
| 复杂 | Opus | 架构级改动、复杂算法、安全敏感 |

用 `AskUserQuestion` 让用户确认或调整模型分配。

3T.2 创建 Team + spawn Builders

按 Layer 分组 spawn Builder teammates。严格规则：

- 同一 Layer 的 Builder 在**一条消息**中并行 spawn（多个 Task 调用）
- 每个 Task 调用**必须**包含完整的 name、model、prompt、subagent_type、mode 参数，禁止发送空 Task
- 当前 Layer 全部 Builder 完成前，**禁止** spawn 下一 Layer
- 用 `TaskOutput` 逐个等待当前 Layer 的所有 Builder，确认全部 completed 后再 spawn 下一 Layer

spawn 语法（每个 Builder 必须严格遵循）：

```
Task({
  name: "Builder-<N>",
  description: "<简短描述>",
  model: "<haiku|sonnet|opus>",
  prompt: "<完整 prompt，见下方模板>",
  subagent_type: "general-purpose",
  mode: "bypassPermissions"
})
```

Builder prompt 内容：

```
你是 Builder，负责实施一个子任务。严格按照以下指令执行。

## 你的任务
<从计划中提取该 Builder 负责的子任务全部内容，包括实施步骤>

## 工作目录
{{WORKDIR}}

## 文件范围约束（硬性规则）
你只能创建或修改以下文件：
<文件列表>
严禁修改任何其他文件。违反此规则等于任务失败。

## 多模型咨询（可选）
遇到拿不准的逻辑时，可调用 codeagent-wrapper 咨询：
~/.claude/bin/codeagent-wrapper --backend <codex|gemini> - "{{WORKDIR}}" <<'EOF'
<咨询内容>
OUTPUT: Unified Diff Patch ONLY.
EOF
军师输出仅供参考，你自己决定最终代码。

## 实施要求
1. 严格按照实施步骤执行
2. 代码必须符合项目现有规范
3. 完成后运行相关的 lint/typecheck 验证
4. 代码应自解释，非必要不加注释

## 验收标准
<从计划中提取>

完成所有步骤后，标记任务为 completed。
```

依赖关系：严格按 Layer 顺序执行。Layer N 全部 Builder completed 后，才能 spawn Layer N+1。spawn 完成后，Lead（Opus）进入 delegate 模式，只协调不写码。

3T.3 监控进度

按 Layer 逐层等待：

```
# Layer 1 全部 spawn 后，逐个等待
TaskOutput({ task_id: "<builder-1-id>", block: true, timeout: 600000 })
TaskOutput({ task_id: "<builder-2-id>", block: true, timeout: 600000 })
# 确认全部 completed → spawn Layer 2
# 重复直到所有 Layer 完成
```

- Builder 求助 → Lead 分析问题给指导，不替它写码
- Builder 失败 → 记录原因，不影响其他 Builder
- 10 分钟超时后继续轮询，绝对不要 Kill 进程

3T.4 汇总

所有 Builder 完成后，汇总变更清单，关闭 Team。

---

### Phase 4：审查

4.1 Claude 先自审

Claude 先检查自己（或 Builders）的产出：是否符合计划、是否引入副作用、代码质量是否达标。

4.2 咨询军师审查（并行）

在同一条消息中发起两个 Bash 调用：

1. Codex 审查（`run_in_background: true`，优先 `resume CODEX_SESSION`）：
   - ROLE_FILE: `~/.claude/.ccg/prompts/codex/reviewer.md`
   - 输入：变更的 Diff + 目标文件
   - 关注：安全性、性能、错误处理、逻辑正确性

2. Gemini 审查（`run_in_background: true`，优先 `resume GEMINI_SESSION`）：
   - ROLE_FILE: `~/.claude/.ccg/prompts/gemini/reviewer.md`
   - 输入：变更的 Diff + 目标文件
   - 关注：可访问性、设计一致性、用户体验

用 `TaskOutput` 等待两个模型的完整审查结果。

4.3 Claude 终审裁决

1. 综合 Claude 自审 + 双军师审查意见
2. 按严重性分级：Critical（必须修复）/ Warning（建议修复）/ Info（可选）
3. 按信任规则裁决分歧：后端听 Codex，前端听 Gemini
4. 不采纳的意见须说明理由
5. 执行必要修复
6. Critical > 0 时重复 4.2 直到 Critical = 0

4.4 交付报告

向用户报告：变更摘要（文件表）、三方审查结果、采纳决策、后续建议。

---

## 关键规则

1. Claude 主导 - 每个环节 Claude 先想，再问军师，最后 Claude 定
2. 咨询义务 - 关键节点必须咨询，不采纳须给理由
3. 代码主权 - 所有文件修改由 Claude 或 Builder 执行，军师零写入权限
4. SESSION_ID 贯穿 - 从 Phase 1 到 Phase 4，军师的上下文不断
5. 信任规则 - 后端分歧听 Codex，前端分歧听 Gemini
6. Lead 不写码 - M/L 级一律 spawn Builder，Lead 上下文留给决策
7. Builder 模型按需 - 简单 Haiku，中等 Sonnet，复杂 Opus
