---
name: vibe-coding-workflow
description: Multi-model vibe coding workflow with 3 levels (Lite / Standard / Strict). User picks level, Hermes writes plan, models execute in sequence, Codex reviews. No execution before user confirms plan. Codex may take over implementation only when the primary writer is unavailable.
author: Hermes
tags:
  - vibe-coding
  - workflow
  - multi-model
  - opencode
  - codex
  - qwen
created: 2026-04-19
---

# Vibe Coding Workflow

## Core Principle

**User describes需求 → Hermes offers two paths → User picks → Plan sent to user for review → User confirms → Pipeline executes.**

### 初始路径选择

收到需求后，Hermes 询问用户：

```
🔧 [项目名] — 请选择路径：

  1️⃣ 拆解模式（M2.7 先拆解子任务，再逐步执行）
      适用：复杂需求、不确定实现方案、需要全局规划

  2️⃣ 直达模式（Qwen 直接写 Plan，跳过拆解）
      适用：hotfix、小改动、需求已清晰

输入序号或说『1』『2』选择。
```

**路径选择等待策略：**

如果用户暂未选择路径，Hermes 可以给出默认建议，但不要把它伪装成已经得到用户确认：

| 需求特征 | 默认建议 |
|---------|---------|
| 提到 fix/hotfix/patch/bug | 直达模式 |
| 提到 重构/架构/重构/大规模 | 拆解模式 |
| 需求描述 ≥ 3 句 | 拆解模式 |
| 需求描述 ≤ 2 句且无复杂关键字 | 直达模式 |
| 跨多个文件/模块 | 拆解模式 |
| 单文件/简单修改 | 直达模式 |

Models are tools. Pick the right tool for the job — don't force every project through the full pipeline.

## ⚠️ Pitfalls（必须避免）

### Codex review 命令语法
**禁止这样用：**
\`\`\`bash
codex review --uncommitted '<prompt>'
\`\`\`
这会报 `the argument '--uncommitted' cannot be used with '[PROMPT]'`，因为 `--uncommitted` 和 `<PROMPT>` 互相排斥。

**正确用法（两选一）：**

**Mode A — 无额外提示（推荐）：**
\`\`\`bash
codex review --uncommitted
\`\`\`

**Mode B — 有额外提示（覆盖 staged + unstaged）：**
\`\`\`bash
git diff HEAD -- . | codex exec - "Review this diff. Focus on <extra context>. Return only concrete findings."
\`\`\`
注意：
- 用管道传 diff 内容，**不要**用 `codex exec - < file`（stdin 方式 Codex 不接受）
- `git diff HEAD` 覆盖 staged + unstaged，不包括未跟踪文件；如果本次改动新增了文件，优先使用 `codex review --uncommitted`
- review 结果只展示，不自动写入文件（sandbox 有 workspace-write 权限但不要依赖它）

### 通用规则
- 默认情况下，Codex 用于 review，不承担主写职责
- Qwen 3.6 Plus 是默认主写模型，Codex 是默认复核模型
- 仅当 OpenCode 不可用或额度耗尽时，Codex 才接管主写
- 所有代码必须等用户确认 Plan 后才能执行

---

## Model Roles

| 模型 | 职责 | 使用场景 |
|---|---|---|
| **MiniMax M2.7** | 拆解任务、列子任务清单 | 仅做任务分解，不写代码 |
| **Qwen 3.6 Plus** | Plan 写作 + 主写（样板 + 核心模块） | Level 1+，OpenCode 额度充足 |
| **Codex CLI 0.121** | 复核 + 审核 + 备用主写 | 全部级别；OpenCode 耗尽时接管主写 |

> **默认分工：复核 ≠ 主写。** Qwen 3.6 Plus 负责写代码，Codex 负责复核和审核。只有在 OpenCode 不可用或额度耗尽时，Codex 才接管主写，两者职责临时合并。

### 额度切换规则（按顺序 fallback）

每次任务启动前检查：

```
OpenCode 额度充足 → Qwen 3.6 Plus 主写
OpenCode 额度不足 → minimax-m2.7 主写（OpenCode）
OpenCode 耗尽 → Codex 接管主写
Codex 也耗尽 → Hermes MiniMax Provider fallback
```

---

## Level 1 · Lite

**适用：hotfix、小改动、patch、低风险修改（80% 项目走这个）**

### 路径差异

- **拆解模式**：M2.7 拆解 → Qwen 写代码 → Codex 复核 → 完成
- **直达模式**：Qwen 直接写代码 → Codex 复核 → 完成（跳过拆解）

### Steps（拆解模式）
1. **M2.7 拆解** → 子任务清单（发用户确认）
2. **Qwen 3.6 Plus** → 写 Plan（发用户确认 Plan）
3. **Qwen 3.6 Plus** → 写代码（用户确认 Plan 后再执行）
4. **Codex** → 增量复核（只 review 改动文件）
5. **Commit**

### Steps（直达模式）
1. **Qwen 3.6 Plus** → 写 Plan（发用户确认 Plan）
2. **Qwen 3.6 Plus** → 写代码（用户确认 Plan 后再执行）
3. **Codex** → 增量复核（只 review 改动文件）
4. **Commit**

### 命令参考
```bash
# 检查额度
opencode stats

# 主写（Qwen，OpenCode 额度充足）
opencode run '[子任务]' --model opencode-go/qwen3.6-plus

# 主写（OpenCode 额度不足，切换 minimax-m2.7）
opencode run '[子任务]' --model opencode-go/minimax-m2.7

# OpenCode 耗尽时，切换 Codex 主写
codex exec --full-auto 'Write code for: [任务]. Implement fully.'

# Codex 增量复核（当前工作区未提交改动）
codex review --uncommitted

# 全量最终审核（相对默认分支）
codex review --base "$(git symbolic-ref --short refs/remotes/origin/HEAD | sed 's#^origin/##')"
```

---

## Level 2 · Standard

**适用：常规功能开发、跨文件改动、架构小调整**

### 路径差异

- **拆解模式**：M2.7 拆解 → Qwen 写 Plan + 样板 + 核心 → Codex 收尾复核 → Codex 最终审核
- **直达模式**：Qwen 直接写 Plan + 样板 + 核心 → Codex 收尾复核 → Codex 最终审核（跳过拆解）

### 并行主写

Level 2 可选并行模式（无依赖模块时启用）：

```
Qwen 写模块 A（后台） + Qwen 写模块 B（后台）→ 合并 → Codex 复核
```

### Steps（拆解模式）
1. **M2.7 拆解** → 子任务清单（发用户确认）
2. **Qwen 3.6 Plus** → 写完整 Plan（发用户确认）
3. **Qwen 3.6 Plus** → 样板代码 / 基础结构（用户确认 Plan 后再执行）
4. **Qwen 3.6 Plus** → 核心模块 / 关键函数
5. **Codex** → 增量复核（只 review 改动文件）
6. **Codex** → 全量最终审核
7. **Commit**

### Steps（直达模式）
1. **Qwen 3.6 Plus** → 直接写完整 Plan（发用户确认）
2. **Qwen 3.6 Plus** → 样板代码 / 基础结构（用户确认 Plan 后再执行）
3. **Qwen 3.6 Plus** → 核心模块 / 关键函数
4. **Codex** → 增量复核（只 review 改动文件）
5. **Codex** → 全量最终审核
6. **Commit**

### 命令参考
```bash
# 检查额度
opencode stats

# Plan 写作（Qwen，OpenCode 额度充足）
opencode run '写 Plan：[需求]' --model opencode-go/qwen3.6-plus

# 核心逻辑（Qwen 主写）
opencode run '写核心：[子任务]' --model opencode-go/qwen3.6-plus
# 额度不足时，切换 minimax-m2.7
opencode run '写核心：[子任务]' --model opencode-go/minimax-m2.7
# OpenCode 耗尽时，切换 Codex 主写
codex exec --full-auto 'Write code for: [任务]. Implement fully.'

# 增量复核（当前工作区未提交改动）
codex review --uncommitted

# 全量最终审核（相对默认分支）
codex review --base "$(git symbolic-ref --short refs/remotes/origin/HEAD | sed 's#^origin/##')"
```

### 并行主写

Level 2+ 可选并行模式（无依赖模块时启用）。每个并行任务必须在独立 worktree 或独立仓库副本中工作，不要在同一个工作目录里后台并行改代码：

```bash
# 为不同模块创建独立 worktree
git worktree add ../module-a -b feature/module-a
git worktree add ../module-b -b feature/module-b

# 在各自 worktree 中分别执行
cd ../module-a
opencode run '写模块 A：[子任务 A]' --model opencode-go/qwen3.6-plus

cd ../module-b
opencode run '写模块 B：[子任务 B]' --model opencode-go/qwen3.6-plus

# 合并前分别 review，再回到主仓库合并
cd -
codex review --base "$(git symbolic-ref --short refs/remotes/origin/HEAD | sed 's#^origin/##')"
```

**适用场景**：模块之间无依赖、可独立开发的功能
**不适用**：有共享类型/接口的模块，或需要频繁同步共享上下文的模块
**合并前**：每个分支必须先完成自检和 Codex review，再执行 merge

---

## Level 3 · Strict

**适用：大架构调整、长期维护项目、跨模块重构、上线前高要求审查**

### 路径差异

- **拆解模式**：M2.7 拆解 → Qwen 写 Plan → Qwen 样板 → Qwen 核心 → Codex 中期复核 → 修复 → 测试验证 → 回归检查 → Codex 最终过审
- **直达模式**：Qwen 直接写 Plan → Qwen 样板 → Qwen 核心 → Codex 中期复核 → 修复 → 测试验证 → 回归检查 → Codex 最终过审（跳过拆解）

### 并行主写

Level 3 可选并行模式（无依赖模块时启用）：

```
Qwen 写模块 A（后台） + Qwen 写模块 B（后台） + Qwen 写模块 C（后台）→ 合并 → Codex 复核
```

所有并行任务完成后，统一做一次增量中期复核。

### Steps（拆解模式）
1. **M2.7 拆解** → 详细子任务清单（发用户确认）
2. **Qwen 3.6 Plus** → 完整 Plan，含文件清单、风险点（发用户确认）
3. **Qwen 3.6 Plus** → 样板代码 / 基础结构（用户确认 Plan 后再执行）
4. **Qwen 3.6 Plus** → 核心模块 / 复杂逻辑
5. **Codex** → 增量中期复核（只 review 改动文件）
6. **Codex** → 修复验证
7. **测试 + 回归验证**
8. **Codex** → 全量最终审核
9. **Commit**

### Steps（直达模式）
1. **Qwen 3.6 Plus** → 直接写完整 Plan，含文件清单、风险点（发用户确认）
2. **Qwen 3.6 Plus** → 样板代码 / 基础结构（用户确认 Plan 后再执行）
3. **Qwen 3.6 Plus** → 核心模块 / 复杂逻辑
4. **Codex** → 增量中期复核（只 review 改动文件）
5. **Codex** → 修复验证
6. **测试 + 回归验证**
7. **Codex** → 全量最终审核
8. **Commit**

### 命令参考
```bash
# 检查额度
opencode stats

# 完整 Plan（Qwen，OpenCode 额度充足）
opencode run '写详细 Plan：[需求]' --model opencode-go/qwen3.6-plus

# 样板代码（Qwen 主写）
opencode run '写样板：[子任务]' --model opencode-go/qwen3.6-plus

# 核心逻辑（Qwen 主写）
opencode run '写核心：[子任务]' --model opencode-go/qwen3.6-plus
# 额度不足时，切换 minimax-m2.7
opencode run '写核心：[子任务]' --model opencode-go/minimax-m2.7
# OpenCode 耗尽时，切换 Codex 主写
codex exec --full-auto 'Write code for: [任务]. Implement fully.'

# 中期复核（当前工作区未提交改动）
codex review --uncommitted

# 修复验证（当前工作区未提交改动）
codex review --uncommitted

# 最终审核（相对默认分支）
codex review --base "$(git symbolic-ref --short refs/remotes/origin/HEAD | sed 's#^origin/##')"
```

---

## Quota Monitoring

**检查命令：**
```bash
# OpenCode 额度
opencode stats

# Codex 额度（查看日志或账号余额）
```

**规则：**
- 每次任务启动前检查 OpenCode 额度，决定主写模型
- OpenCode 额度耗尽时，Codex 接管主写（复核职责合并）
- Codex 额度也耗尽时，Hermes 使用内置 MiniMax Provider 完成剩余工作
- 每次切换时通知用户
- 如果无法可靠读取额度信息，就显式告知用户当前采用的模型，不要假装已经完成检查

---

## Plan 审核规则（所有级别）

**Plan 完成后必须推送用户，等确认再执行，这是硬规则。**

发送格式：
```
📋 Plan 审核 — [项目名]

[完整 Plan 内容]

请确认后我再执行。输入『可以』『同意』『开始』立即开工。
```

### Plan 修订循环

用户对 Plan 有意见时（说"改一下"、"不满意"、"重新写"等）：

1. **收集反馈** → 整理用户的修改意见
2. **Qwen 重新生成** → `opencode run '根据反馈重新生成 Plan：[修改意见]' --model opencode-go/qwen3.6-plus`
3. **重新发送审核** → 重复 Plan 审核格式
4. **循环直到用户满意或中止**

**Plan 审核等待策略：**

如果一段时间内未收到确认，Hermes 只做提醒，不自动执行代码。可以按风险等级给出下一步建议：

| 情况 | 处理方式 |
|------|---------|
| 低风险改动（Level 1） | 发送提醒，告知用户当前状态，继续等待 |
| 中高风险改动（Level 2+） | 发送提醒，说明尚未执行，等待用户确认 |
| 阻塞性问题 | 明确说明阻塞原因，暂停任务，等待用户决策 |

**等待期间不执行任何代码。**

### Review 反馈循环

Codex 复核失败时：

1. **Codex 报告问题** → Hermes 提取问题清单
2. **通知用户** → 说明问题严重程度（阻塞 / 建议修改）
3. **用户决策**：
   - "修复" → Qwen 根据意见修复 → Codex 重新复核
   - "跳过" → 记录风险，继续（仅 Level 1 适用）
   - "中止" → 停止任务，保留改动

**注意**：阻塞性问题（安全漏洞、数据丢失风险）必须修复，不能跳过。

**Review 反馈等待策略：**

Codex 复核失败而用户暂未决策时：

| 问题严重程度 | 处理方式 |
|------------|---------|
| 阻塞性问题（安全漏洞、数据丢失风险） | 立即停止，保留改动，通知用户，**不自动回滚** |
| 建议修改（非阻塞） | 等待用户决策，并明确当前风险 |
| Level 3 严格审查 | 暂停后续步骤，等待用户明确放行 |

**禁止自动回滚。** 回滚是破坏性操作，必须由用户明确授权。

---

## Project Path

> 获取方式：优先使用 `git rev-parse --show-toplevel`，不在 git 仓库中时再用 `pwd`

执行 vibe-coding 任务前，通过 `cd [项目路径]` 切换到目标仓库根目录。

---

## What Hermes Never Does

- ❌ Does NOT read files aimlessly or exhaustively without a task-specific reason
- ❌ Does NOT skip the review step
- ❌ Does NOT commit without Codex review approval
- ❌ **Does NOT execute any Plan before user explicitly confirms**
- ❌ **Does NOT start coding until user says "可以"/"同意"/"开始"**
- ❌ Does NOT use any model other than Codex for review
- ❌ Does NOT auto-rollback without explicit user authorization
- ❌ Does NOT auto-execute code while waiting for user confirmation
