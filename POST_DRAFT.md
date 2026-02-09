# 帖子标题

**【开源·更新】CCG v1.7.61 : Claude Code 编排三 CLI 协作 | Codex + Gemini + Claude | Agent Teams 并行实施**

---

# 正文

> **GitHub**: https://github.com/fengshao1227/ccg-workflow
> 觉得好用请留下你的 ⭐ Star

[![npm version](https://img.shields.io/npm/v/ccg-workflow.svg)](https://www.npmjs.com/package/ccg-workflow) [![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](https://opensource.org/licenses/MIT) [![Claude Code](https://img.shields.io/badge/Claude%20Code-Compatible-green.svg)](https://claude.ai/code)

## 一句话介绍

**CCG** = Claude Code 编排 Codex + Gemini 三 CLI 协作

- 前端任务 → **Gemini**（擅长 UI/CSS/组件）
- 后端任务 → **Codex**（擅长逻辑/算法/调试）
- 全栈整合 → **Claude**（编排决策、质量把控、代码审核）

外部模型无写入权限，仅返回 Patch，由 Claude 审核后应用。

---

## 🚀 30 秒安装

```bash
npx ccg-workflow
```

选择「初始化工作流」→ 自动安装 25 个命令 → 重启终端生效

**要求**：Claude Code CLI、Node.js 20+
**可选**：Codex CLI（后端）、Gemini CLI（前端）

> 只有 Claude Code 也能用，就是没有多模型协作了

---

## 📦 版本更新

<details>
<summary><strong>🔥 v1.7.61 — Agent Teams 并行实施（2026-02-10）</strong></summary>

新增独立的 Team 系列命令，利用 Claude Code Agent Teams 实验特性实现多 agent 并行开发：

| 命令 | 用途 |
|------|------|
| `/ccg:team-research` | 需求 → 约束集（并行探索代码库） |
| `/ccg:team-plan` | 约束 → 零决策并行计划 |
| `/ccg:team-exec` | spawn Builder teammates 并行写代码 |
| `/ccg:team-review` | 双模型交叉审查 |

**工作流**：
```
/ccg:team-research → /clear → /ccg:team-plan → /clear → /ccg:team-exec → /clear → /ccg:team-review
```

**特点**：
- 每步 `/clear` 隔离上下文，通过文件传递状态，不怕上下文爆
- Builder teammates 使用 Sonnet 并行写代码，速度翻倍
- 完全独立体系，不依赖现有 `/ccg:workflow` 等命令

**前置条件**：需手动启用 Agent Teams：
```json
// ~/.claude/settings.json
{ "env": { "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1" } }
```

**实战效果**：一个实时协作看板 API（JWT 认证 + 4 层嵌套 CRUD + WebSocket + 权限控制 + 拖拽排序），10 个源文件、20+ REST 端点，全链路验证零 bug。

</details>

<details>
<summary><strong>v1.7.59 — 内置 Prompt 增强（2026-02-09）</strong></summary>

- 新增 `/ccg:enhance` 命令，内置 Prompt 增强
- 移除对 ace-tool `enhance_prompt` 的依赖

</details>

<details>
<summary><strong>v1.7.58 — 输出风格配置（2026-02-09）</strong></summary>

- **6 种输出风格**：默认 / 专业工程师 / 猫娘 / 老王 / 大小姐 / 邪修

</details>

<details>
<summary><strong>v1.7.57 — MCP 扩展 + API 配置 + 实用工具（2026-02-08）</strong></summary>

- **MCP 扩展**：ContextWeaver（推荐免费）+ 辅助工具（Context7/Playwright/DeepWiki/Exa）
- **API 配置**：支持自定义 URL + Key，自动优化配置
- **实用工具**：ccusage 用量分析 + CCometixLine 状态栏
- **Claude Code 安装**：npm / homebrew / curl / powershell / cmd
- **OpenSpec 集成**：规范驱动开发，把需求变成约束

</details>

<details>
<summary><strong>v1.7.52 — OpenSpec OPSX 架构（2026-01-26）</strong></summary>

- 迁移到 OPSX 架构，废弃 `/openspec:xxx`，启用 `/opsx:xxx`
- 更新 `spec-*` 系列命令

</details>

---

## ✨ 核心特性

| 特性 | 说明 |
|-----|------|
| **固定路由** | 前端→Gemini，后端→Codex，全栈→Claude |
| **多模型并行** | Codex ∥ Gemini 同时调用，交叉验证 |
| **Agent Teams** | spawn Builder teammates 并行写代码（v1.7.61+） |
| **6阶段工作流** | Prompt增强 → 检索 → 分析 → 原型 → 实施 → 审计 |
| **25个斜杠命令** | 开发工作流 + Git 工具 + OPSX + Agent Teams |
| **13个专家提示词** | Codex 6 个 + Gemini 7 个，零 token 动态注入 |
| **6种输出风格** | 默认 / 专业工程师 / 猫娘 / 老王 / 大小姐 / 邪修 |
| **跨平台支持** | macOS / Linux / Windows × Intel / ARM |

### 架构图

```
Claude Code (编排)
       │
   ┌───┴───┐
   ↓       ↓
Codex   Gemini
(后端)   (前端)
   │       │
   └───┬───┘
       ↓
  Unified Patch
 (Claude 审核后应用)
```

---

## 📚 命令速查

<details>
<summary><strong>🔧 日常开发（6 个）</strong></summary>

```bash
/ccg:workflow   # 完整6阶段流程
/ccg:plan       # 多模型规划，生成计划
/ccg:execute    # 多模型执行，执行计划
/ccg:feat       # 新功能开发
/ccg:frontend   # 纯前端，Gemini 快速模式
/ccg:backend    # 纯后端，Codex 快速模式
```

</details>

<details>
<summary><strong>🔍 分析调试（5 个）</strong></summary>

```bash
/ccg:analyze    # 技术分析
/ccg:debug      # 问题诊断+修复
/ccg:optimize   # 性能优化
/ccg:test       # 测试生成
/ccg:review     # 代码审查（无参数自动审查 git diff）
```

</details>

<details>
<summary><strong>🔀 Git 工具（4 个）</strong></summary>

```bash
/ccg:commit         # 智能提交（conventional commit）
/ccg:rollback       # 交互式回滚
/ccg:clean-branches # 清理已合并分支
/ccg:worktree       # Worktree 管理
```

</details>

<details>
<summary><strong>📋 OpenSpec 规范驱动（5 个）</strong></summary>

集成了 [OpenSpec (OPSX)](https://github.com/fission-ai/opsx)，把需求变成约束，让 AI 没法自由发挥。

| 命令 | 干嘛的 |
|------|--------|
| `/ccg:spec-init` | 初始化 OpenSpec 环境 |
| `/ccg:spec-research` | 分析需求，输出约束集 |
| `/ccg:spec-plan` | Codex + Gemini 并行分析，生成执行计划 |
| `/ccg:spec-impl` | 按计划一步步实现，完了自动归档 |
| `/ccg:spec-review` | 双模型审查，随时可以用 |

```
需求 ──→ spec-research ──→ spec-plan ──→ spec-impl
              │                │              │
           约束集          零决策计划      机械执行
```

每个阶段之间可以 `/clear`，状态存在 `openspec/` 目录，不怕上下文爆。

</details>

<details>
<summary><strong>⚡ Agent Teams 并行实施（4 个）— v1.7.61 新增</strong></summary>

利用 Claude Code Agent Teams 实验特性，spawn 多个 Builder teammates 并行写代码。

| 命令 | 干嘛的 |
|------|--------|
| `/ccg:team-research` | 并行探索代码库，产出约束集 |
| `/ccg:team-plan` | Codex + Gemini 分析，拆分为并行子任务 |
| `/ccg:team-exec` | spawn Builder teammates 并行实施 |
| `/ccg:team-review` | 双模型交叉审查 |

```
需求 ──→ team-research ──→ team-plan ──→ team-exec ──→ team-review
              │                │              │              │
           约束集          并行计划     Builder×N 并行    双模型审查
```

**vs 传统工作流**：每步 `/clear` 隔离上下文，Builder 并行实施。适合可拆分为 3+ 独立模块的任务。

**前置条件**：
```json
// ~/.claude/settings.json
{ "env": { "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1" } }
```

</details>

<details>
<summary><strong>📁 项目管理（1 个）</strong></summary>

```bash
/ccg:init       # 初始化项目 CLAUDE.md
```

</details>

---

## 🔧 交互式菜单

```bash
npx ccg-workflow
```

<details>
<summary><strong>菜单选项详情</strong></summary>

- 初始化工作流
- 更新工作流
- 配置 MCP
- 配置 API
- 配置输出风格
- 实用工具
- 安装 Claude Code
- 卸载工作流

### 配置 MCP

**代码检索 MCP（二选一）**：
- ✅ **ContextWeaver**（推荐）- 本地混合搜索，需要硅基流动 API Key（免费）
- ⚠️ **ace-tool**（收费）- Augment 官方

**辅助工具 MCP（可选）**：
- **Context7** - 获取最新库文档
- **Playwright** - 浏览器自动化/测试
- **DeepWiki** - 知识库查询
- **Exa** - 搜索引擎（需 API Key，有免费额度）

### 配置 API

- 支持自定义 `ANTHROPIC_BASE_URL` 和 `ANTHROPIC_API_KEY`
- 自动添加优化配置（禁用遥测、MCP 超时等）
- 自动添加 codeagent-wrapper 权限白名单

### 配置输出风格

| 风格 | 说明 |
|-----|------|
| 默认 | Claude Code 原生风格 |
| 专业工程师 | 简洁专业的技术风格 |
| 猫娘工程师 | 可爱猫娘语气喵~ |
| 老王工程师 | 接地气的老王风格 |
| 大小姐工程师 | 优雅大小姐语气 |
| 邪修风格 | 宿命深渊·道语标签 |

### 实用工具

- **ccusage** - Claude Code 用量分析
- **CCometixLine** - 状态栏工具（Git + 用量跟踪）

### 安装 Claude Code

支持多种安装方式：npm / homebrew / curl / powershell / cmd

</details>

---

## ❓ 常见问题

<details>
<summary><strong>Q: codeagent-wrapper: command not found？</strong></summary>

PATH 未生效，重启终端或执行：
```bash
source ~/.zshrc  # Mac/Linux
```
</details>

<details>
<summary><strong>Q: 如何更新？</strong></summary>

```bash
npx ccg-workflow@latest
# 选择 "更新工作流"
```
</details>

<details>
<summary><strong>Q: 如何卸载？</strong></summary>

```bash
npx ccg-workflow
# 选择 "卸载工作流"
```
</details>

<details>
<summary><strong>Q: Codex 任务卡住？</strong></summary>

Codex CLI 0.80.0 已知 bug，设置环境变量解决：
```bash
export CODEAGENT_POST_MESSAGE_DELAY=1
```
或在 `~/.claude/settings.json` 中配置：
```json
{ "env": { "CODEAGENT_POST_MESSAGE_DELAY": "1" } }
```
</details>

<details>
<summary><strong>Q: Claude Code 任务超时？</strong></summary>

修改 `~/.claude/settings.json`：
```json
{
  "env": {
    "CODEX_TIMEOUT": "7200",
    "BASH_DEFAULT_TIMEOUT_MS": "600000",
    "BASH_MAX_TIMEOUT_MS": "3600000"
  }
}
```
</details>

<details>
<summary><strong>Q: Agent Teams 怎么启用？</strong></summary>

在 `~/.claude/settings.json` 中添加：
```json
{ "env": { "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1" } }
```
然后重新运行 `npx ccg-workflow` 安装最新命令。
</details>

<details>
<summary><strong>Q: OpenSpec CLI 装不上？</strong></summary>

```bash
npm install -g @fission-ai/openspec@latest
```
</details>

---

## 🙏 鸣谢

- [cexll/myclaude](https://github.com/cexll/myclaude) - codeagent-wrapper
- [UfoMiao/zcf](https://github.com/UfoMiao/zcf) - Git 工具 + 输出风格
- [telagod/code-abyss](https://github.com/telagod/code-abyss) - 邪修输出风格
- [GudaStudio/skills](https://github.com/GuDaStudio/skills) - 智能路由设计
- [fission-ai/opsx](https://github.com/fission-ai/opsx) - OpenSpec 规范驱动框架
- [ace-tool MCP](https://linux.do/t/topic/1344562) - 代码检索工具

---

**版本**: v1.7.61 | [GitHub Issues](https://github.com/fengshao1227/ccg-workflow/issues)
