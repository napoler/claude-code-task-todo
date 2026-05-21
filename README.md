# Todo Guard Hooks 使用指南

## 概述

这两个 JavaScript Hook 用于 Claude Code 的 PreToolUse 机制，实现：
1. **todo-guard.js** - 强制任务检查 + 自动拆分
2. **todo-session-restore.js** - 会话恢复 + 归档

## todo-sync-wiki.js

### 功能

**PostToolUse hook**：TaskCreate/TaskUpdate 时自动同步到 wiki

- TaskCreate → 新增任务到 `todo-registry.md`
- TaskUpdate → 更新匹配的任务状态

### wiki格式

```markdown
## 当前任务
- [pending] 📋 任务名称 @2026-05-22 00:00:00 [sessionId]
- [completed] ✅ 完成任务 @2026-05-15 00:00:00

## 历史任务
（7天前自动归档）
```

## 安装

在 `~/.claude/settings.json` 中注册：

```json
{
  "hooks": {
    "PreToolUse": {
      "todo-guard": "node /path/to/todo-guard.js"
    },
    "PostToolUse": {
      "todo-sync-wiki": "node /path/to/todo-sync-wiki.js"
    },
    "SessionStart": {
      "todo-session-restore": "node /path/to/todo-session-restore.js"
    }
  }
}
```

## 文件结构

```
hooks/
├── todo-guard.js            # 任务检查 + 拆分
├── todo-session-restore.js  # 会话恢复 + 归档
├── todo-sync-wiki.js       # wiki同步
└── README.md               # 本文档
```

### 功能一：强制任务检查

**规则**：所有非只读操作必须先有 pending/in_progress 任务

| 工具类型 | 示例 | 处理 |
|---------|------|------|
| 只读工具 | Read, Glob, Grep, WebFetch, WebSearch | 直接放行 |
| 任务管理 | TaskCreate, TaskUpdate, TaskStop, TaskGet, TaskList | 白名单放行 |
| 写操作 | Write, Edit, Bash, Agent, SendMessage | 需有 active 任务 |

**无任务时的响应**：
```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "additionalContext": "⚠️ TODO GUARD: 无active任务禁止执行写操作",
    "blockAction": true
  }
}
```

### 功能二：自动任务拆分

**所有 TaskCreate 都触发拆分**，返回 splitTask 对象：

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "additionalContext": "📋 任务拆分(通用模板)\n\n任务: \"xxx\"\n\n拆分为 4 个子任务:...",
    "blockAction": false,
    "splitTask": {
      "parentSubject": "原始任务名",
      "subTasks": [
        {"subject": "Analyze: xxx", "status": "pending"},
        {"subject": "Plan: xxx", "status": "pending"},
        {"subject": "Execute: xxx", "status": "pending"},
        {"subject": "Verify: xxx", "status": "pending"}
      ],
      "mode": "auto"
    }
  }
}
```

### 拆分模式

| 模式 | 触发条件 | 示例 |
|-----|---------|------|
| phase | `/^Phase (\d+):/i` | `Phase 3: X` → Phase 1/2/3 |
| pipeline | `/rewrite pipeline\|fix rewrite/i` | → Analyze/Rewrite/Verify |
| auditFix | `/audit.*rewrite\|rewrite.*audit/i` | → Audit/Rewrite/Verify |
| qualityFix | `/quality.*fix.*rewrite/i` | → Audit/Rewrite/Verify |
| auto | 无匹配 | → Analyze/Plan/Execute/Verify |

## todo-session-restore.js

### 功能

1. **会话恢复**：从 `~/.omc/wiki/todo-registry.md` 读取任务
2. **归档**：自动归档超过7天的已完成任务到 `## 历史任务` 区段

### 任务格式

```
- [pending] 📋 任务名称 @2026-05-22 00:00:00 [sessionId]
- [completed] ✅ 完成任务 @2026-05-15 00:00:00
```

### 输出

```
OMC_HOOK_TASK_RESTORE {"type":"hook_instruction","hook":"todo-session-restore","action":"restore_tasks","count":N,"tasks":[...]}
```

## 只读Bash命令白名单

```
ls, find -type f, cat, head, tail, wc,
git status|log|diff|show|branch,
rg, grep, fd, ripgrep
```

## 环境变量

| 变量 | 默认值 | 说明 |
|-----|-------|------|
| CLAUDE_CONFIG_DIR | ~/.claude | 配置目录 |
| CLAUDE_SESSION_ID | - | 会话ID |
| CLAUDE_CODE_SESSION_ID | - | SessionStart用 |

## 文件结构

```
hooks/
├── todo-guard.js           # 任务检查 + 拆分
├── todo-session-restore.js  # 会话恢复 + 归档
└── README.md               # 本文档
```