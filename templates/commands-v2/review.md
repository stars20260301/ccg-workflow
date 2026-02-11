---
description: 'CCG 代码审查 - Claude 主导，Codex+Gemini 并行审查，交叉验证'
---

# Review - CCG 代码审查

双模型并行审查，Claude 交叉验证终审。无参数时自动审查当前 git 变更。

$ARGUMENTS

---

## 协作模型

Claude 是主导者（先自审，再听军师，最后终审裁决），Codex 是后端军师（安全、性能、逻辑、错误处理），Gemini 是前端军师（可访问性、设计一致性、UX、模式）。

---

## 核心协议

- 语言协议：与工具/模型交互用英语，与用户交互用中文
- 咨询义务：必须咨询双军师，不采纳须给理由
- 代码主权：外部模型零写入权限；Critical 修复由 Claude 执行
- 信任规则：后端问题听 Codex，前端问题听 Gemini

---

## 多模型调用规范

工作目录：
- `{{WORKDIR}}`：替换为目标工作目录的绝对路径
- 默认使用当前工作目录

调用语法（并行用 `run_in_background: true`）：

```
Bash({
  command: "~/.claude/bin/codeagent-wrapper {{LITE_MODE_FLAG}}--backend <codex|gemini> {{GEMINI_MODEL_FLAG}}- \"{{WORKDIR}}\" <<'EOF'
ROLE_FILE: <角色提示词路径>
<TASK>
审查以下代码变更：
<git diff 内容>
</TASK>
OUTPUT (JSON):
{
  "findings": [
    {
      "severity": "Critical|Warning|Info",
      "dimension": "<审查维度>",
      "file": "path/to/file",
      "line": 42,
      "description": "问题描述",
      "fix_suggestion": "修复建议"
    }
  ],
  "passed_checks": ["已验证的检查项"],
  "summary": "总体评估"
}
EOF",
  run_in_background: true,
  timeout: 3600000,
  description: "简短描述"
})
```

模型参数：`{{GEMINI_MODEL_FLAG}}` 当使用 `--backend gemini` 时替换为 `--gemini-model gemini-3-pro-preview `（注意末尾空格），使用 codex 时替换为空字符串。

角色提示词：

| 模型 | 提示词 |
|------|--------|
| Codex | `~/.claude/.ccg/prompts/codex/reviewer.md` |
| Gemini | `~/.claude/.ccg/prompts/gemini/reviewer.md` |

等待后台任务：

```
TaskOutput({ task_id: "<task_id>", block: true, timeout: 600000 })
```

必须指定 `timeout: 600000`，否则默认只有 30 秒会导致提前超时。若 10 分钟后仍未完成，继续用 `TaskOutput` 轮询，绝对不要 Kill 进程。若因等待时间过长跳过了等待，必须调用 `AskUserQuestion` 询问用户选择继续等待还是 Kill Task。

---

## 执行工作流

### 阶段 1：获取待审查代码

无参数时：执行 `git diff HEAD` 和 `git status --short`。有参数时：使用指定的代码/描述。

调用 `{{MCP_SEARCH_TOOL}}` 获取相关上下文（被修改文件的周边代码）。

### 阶段 2：Claude 先自审

Claude 先独立审查变更：逻辑正确性、是否引入副作用、是否符合项目规范。记录自己发现的问题和不确定的点。

### 阶段 3：咨询军师（并行）

在同一条消息中发起两个 Bash 调用：

1. Codex 后端审查（`run_in_background: true`）：
   - ROLE_FILE: `~/.claude/.ccg/prompts/codex/reviewer.md`
   - 维度：logic / security / performance / error_handling
   - OUTPUT: JSON findings

2. Gemini 前端审查（`run_in_background: true`）：
   - ROLE_FILE: `~/.claude/.ccg/prompts/gemini/reviewer.md`
   - 维度：patterns / maintainability / accessibility / ux / frontend_security
   - OUTPUT: JSON findings

用 `TaskOutput` 等待两个模型的完整审查结果。

### 阶段 4：Claude 终审裁决

1. 综合 Claude 自审 + 双军师审查意见
2. 去重合并重叠问题
3. 按严重性分级：Critical（必须修复）/ Warning（建议修复）/ Info（可选）
4. 按信任规则裁决分歧：后端听 Codex，前端听 Gemini
5. 不采纳的意见须说明理由

### 阶段 5：输出审查报告

向用户报告：审查范围、Critical/Warning/Info 分级问题列表（标注来源 Claude/Codex/Gemini）、已通过检查项、裁决说明、总体评价（代码质量 + 是否可合并）。

### 阶段 6：决策门

- Critical > 0：用 `AskUserQuestion` 询问"立即修复 / 跳过"。选择修复 → Claude 直接修复（后端参考 Codex 建议，前端参考 Gemini 建议），修复后重新运行阶段 3-4 验证，重复直到 Critical = 0。
- Critical = 0：报告通过，建议提交代码。

---

## 关键规则

1. Claude 先审再问 - 不是甩给军师，Claude 自己先有判断
2. 双模型交叉验证 - 捕获单模型审查的盲区
3. 信任规则 - 后端问题听 Codex，前端问题听 Gemini
4. 咨询义务 - 不采纳须给理由
5. Critical 必须清零 - 有 Critical 不能放行
