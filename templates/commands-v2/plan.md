---
description: 'CCG 仅规划 - Claude 主导，咨询 Codex+Gemini，产出零决策实施计划'
---

# Plan - CCG 仅规划

$ARGUMENTS

---

## 协作模型

Claude 是主导者（分析、决策、写计划），Codex 是后端军师（分析、出草案），Gemini 是前端军师（分析、出草案）。本命令仅规划，禁止修改产品代码。产出计划文件供 /ccg:dev 执行。

---

## 核心协议

- 语言协议：与工具/模型交互用英语，与用户交互用中文
- 仅规划：允许读取上下文与写入 `.claude/plan/*`，禁止修改产品代码
- 咨询义务：分析和定稿必须咨询双军师，不采纳须给理由
- 代码主权：外部模型对文件系统零写入权限
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
Bash({
  command: "~/.claude/bin/codeagent-wrapper {{LITE_MODE_FLAG}}--backend <codex|gemini> {{GEMINI_MODEL_FLAG}}- \"{{WORKDIR}}\" <<'EOF'
ROLE_FILE: <角色提示词路径>
<TASK>
需求：<增强后的需求>
上下文：<检索到的项目上下文>
</TASK>
OUTPUT: <期望输出格式>. DO NOT modify any files.
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

SESSION_ID：每次调用返回 `SESSION_ID: xxx`，必须保存并写入计划文件，供 `/ccg:dev` 复用。

等待后台任务（最大超时 600000ms = 10 分钟）：

```
TaskOutput({ task_id: "<task_id>", block: true, timeout: 600000 })
```

必须指定 `timeout: 600000`，否则默认只有 30 秒会导致提前超时。若 10 分钟后仍未完成，继续用 `TaskOutput` 轮询，绝对不要 Kill 进程。若因等待时间过长跳过了等待，必须调用 `AskUserQuestion` 询问用户选择继续等待还是 Kill Task。

---

## 执行工作流

规划任务：$ARGUMENTS

### Phase 1：上下文检索

1.1 Prompt 增强（必须首先执行）

分析 $ARGUMENTS 的意图、缺失信息、隐含假设，补全为结构化需求（明确目标、技术约束、范围边界、验收标准），用增强结果替代原始 $ARGUMENTS。

1.2 上下文检索

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

1.3 需求对齐

若需求仍有模糊空间，必须用 `AskUserQuestion` 向用户输出引导性问题列表，直至需求边界清晰。

---

### Phase 2：分析

2.1 Claude 先形成初步判断

基于检索到的上下文，Claude 先独立思考初步技术方案、预判关键风险、识别需要军师意见的点。

2.2 咨询军师（并行）

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

2.3 Claude 交叉验证

整合 Claude 自身判断 + 双军师意见：

1. 一致点（强信号）→ 直接采纳
2. 分歧点 → 按信任规则裁决：后端听 Codex，前端听 Gemini
3. 互补优势 → 吸收各方独到见解
4. 不采纳的意见 → 记录理由

产出：约束集（硬约束 + 软约束 + 依赖关系 + 风险）

2.4 歧义消解

从约束集和军师意见中提取所有开放问题，用 `AskUserQuestion` 系统性呈现：
- 分组相关问题
- 为每个问题提供上下文
- 在适用时建议默认值

将用户回答转化为额外约束，补入约束集。无开放问题后方可进入 Phase 3。

---

### Phase 3：定稿

3.1 咨询军师出计划草案（并行，推荐）

resume SESSION_ID，让军师在已有上下文基础上出草案：

1. Codex 计划草案（`resume CODEX_SESSION`）：
   - ROLE_FILE: `~/.claude/.ccg/prompts/codex/architect.md`
   - OUTPUT: Step-by-step plan + pseudo-code（重点：数据流/边界条件/错误处理/测试策略）

2. Gemini 计划草案（`resume GEMINI_SESSION`）：
   - ROLE_FILE: `~/.claude/.ccg/prompts/gemini/architect.md`
   - OUTPUT: Step-by-step plan + pseudo-code（重点：信息架构/交互/可访问性/视觉一致性）

3.2 Claude 综合定稿

Claude 参考双方草案，自己写最终计划。计划必须包含：

- 任务类型（前端/后端/全栈）
- 约束集（硬约束 + 软约束）
- 技术方案（说明采纳/不采纳军师意见的理由）
- 实施步骤（每步有预期产物）
- 关键文件表（文件路径:行号 + 操作 + 说明）
- 风险与缓解
- SESSION_ID（CODEX_SESSION + GEMINI_SESSION）

---

### Phase 3 结束：计划交付

`/ccg:plan` 的职责到此结束，必须执行以下动作：

1. 向用户展示完整实施计划
2. 将计划保存至 `.claude/plan/<功能名>.md`
3. 提示用户：计划已保存，可修改或用 `/ccg:dev .claude/plan/<功能名>.md` 执行
4. 立即终止当前回复（Stop here. No more tool calls.）

绝对禁止：
- 问用户 "Y/N" 然后自动执行
- 对产品代码进行任何写操作
- 自动调用 `/ccg:dev` 或任何实施动作

---

## 计划保存

- 首次规划：`.claude/plan/<功能名>.md`
- 迭代版本：`.claude/plan/<功能名>-v2.md`、`-v3.md`...

## 计划修改流程

用户要求修改 → 调整内容 → 更新文件 → 重新展示 → 再次提示审查或执行

---

## 关键规则

1. 仅规划不实施 - 不执行任何代码变更
2. Claude 主导 - Claude 先想，再问军师，最后 Claude 定
3. 咨询义务 - 不采纳军师意见须给理由
4. 约束集思维 - 产出约束而非信息堆砌，收窄解空间
5. SESSION_ID 交接 - 计划末尾必须包含 SESSION_ID 供 `/ccg:dev` 复用
6. 信任规则 - 后端听 Codex，前端听 Gemini
