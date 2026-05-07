# 对话数据存储与记忆机制研究报告 (Phase 10 补充)

> 状态: ✅ 已完成
> 关联 Phase: 10 (会话存储部分)
> 说明: 本文档是 Phase 10 中会话存储和内存管理机制的深入补充

---

## 1. 数据存储位置与方式

### 1.1 存储位置

| 操作系统 | 路径 |
|---------|------|
| Windows | `%APPDATA%/opencode/opencode.db` |
| macOS | `~/Library/Application Support/opencode/opencode.db` |
| Linux | `~/.config/opencode/opencode.db` |

### 1.2 存储方式

- **数据库**: SQLite (WAL 模式)
- **ORM**: Drizzle
- **写入机制**: 事件驱动 (SyncEvent → projectors → DB)
- **事务支持**: 显式事务 + 自动事务

---

## 2. 表结构

### 2.1 Session 表 (会话)

| 字段 | 类型 | 说明 |
|------|------|------|
| id | TEXT (SessionID) | 主键 |
| project_id | TEXT (ProjectID) | 外键 → project.id |
| workspace_id | TEXT (WorkspaceID) | 工作区 ID |
| parent_id | TEXT (SessionID) | 父会话 ID (fork 来源) |
| slug | TEXT | URL 友好 slug |
| directory | TEXT | 工作目录 |
| path | TEXT | 相对路径 |
| title | TEXT | 会话标题 |
| version | TEXT | 协议版本 |
| share_url | TEXT | 分享链接 |
| summary_additions | INTEGER | 代码增加行数 |
| summary_deletions | INTEGER | 代码删除行数 |
| summary_files | INTEGER | 修改文件数 |
| summary_diffs | JSON | 文件 diff |
| revert | JSON | 回滚信息 |
| permission | JSON | 权限规则 |
| agent | TEXT | 当前 Agent |
| model | JSON | 当前模型 {id, providerID, variant} |
| time_created | INTEGER | 创建时间 |
| time_updated | INTEGER | 更新时间 |
| time_compacting | INTEGER | 最后压缩时间 |
| time_archived | INTEGER | 归档时间 |

### 2.2 Message 表 (消息)

| 字段 | 类型 | 说明 |
|------|------|------|
| id | TEXT (MessageID) | 主键 |
| session_id | TEXT (SessionID) | 外键 → session.id |
| time_created | INTEGER | 创建时间 |
| data | JSON | 消息内容 (User/Assistant 信息) |

### 2.3 Part 表 (消息部件)

| 字段 | 类型 | 说明 |
|------|------|------|
| id | TEXT (PartID) | 主键 |
| message_id | TEXT (MessageID) | 外键 → message.id |
| session_id | TEXT (SessionID) | 会话 ID |
| time_created | INTEGER | 创建时间 |
| data | JSON | Part 数据 (TextPart/ToolPart 等) |

### 2.4 Todo 表 (任务列表)

| 字段 | 类型 | 说明 |
|------|------|------|
| session_id | TEXT (SessionID) | 外键 (复合主键 part) |
| content | TEXT | 任务描述 |
| status | TEXT | pending/in_progress/completed/cancelled |
| priority | TEXT | high/medium/low |
| position | INTEGER | 排序位置 (复合主键 part) |
| time_created | INTEGER | 创建时间 |
| time_updated | INTEGER | 更新时间 |

### 2.5 SessionMessage 表 (会话消息 V2)

| 字段 | 类型 | 说明 |
|------|------|------|
| id | TEXT | 主键 |
| session_id | TEXT (SessionID) | 外键 → session.id |
| type | TEXT | user/assistant/shell/compaction/agent-switched/model-switched/synthetic |
| time_created | INTEGER | 创建时间 |
| time_updated | INTEGER | 更新时间 |
| data | JSON | 消息数据 |

---

## 3. 数据格式

### 3.1 消息类型

| 类型 | 说明 | 存储位置 |
|------|------|----------|
| user | 用户输入 | Message + Part |
| assistant | 模型响应 | Message + Part |
| shell | Shell 命令 | Message + Part |
| compaction | 压缩标记 | Message + Part |
| agent-switched | Agent 切换 | SessionMessage |
| model-switched | 模型切换 | SessionMessage |
| synthetic | 系统消息 | SessionMessage |

### 3.2 Part 类型 (message-v2.ts)

| 类型 | 说明 |
|------|------|
| text | 文本内容 |
| tool | 工具调用 |
| reasoning | 推理过程 |
| file | 文件附件 |
| agent | Agent 引用 |
| compaction | 压缩标记 |
| subtask | 子任务 |
| retry | 重试信息 |
| step-start | 步骤开始 |
| step-finish | 步骤结束 |
| snapshot | 快照 |
| patch | 补丁 |

### 3.3 Tool 状态

| 状态 | 说明 |
|------|------|
| pending | 待调用 |
| running | 执行中 |
| completed | 完成 |
| error | 错误 |

---

## 4. 记忆机制 (详细版)

### 4.1 概述

OpenCode 的记忆机制核心是 **上下文压缩 (Compaction)**，当对话长度接近模型上下文限制时自动触发。

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        压缩工作流                                       │
├─────────────────────────────────────────────────────────────────────────┤
│  1. 溢出检测 (isOverflow)                                              │
│     - 检查 tokens.total >= usable                                     │
│                           ↓                                            │
│  2. 创建压缩标记 (create compaction message)                          │
│     - 添加 type="compaction" 的 Part                                   │
│                           ↓                                            │
│  3. 选择保留消息 (select)                                              │
│     - 保留最近 N 轮对话 (tail_turns)                                   │
│     - 保留最近 tokens 预算 (preserve_recent_tokens)                   │
│                           ↓                                            │
│  4. 调用压缩模型生成摘要                                               │
│     - 使用 SUMMARY_TEMPLATE 格式                                       │
│                           ↓                                            │
│  5. 保存摘要 + 更新 tail_start_id                                      │
│                           ↓                                            │
│  6. 清理旧工具输出 (prune)                                             │
│                           ↓                                            │
│  7. 发送 "Continue" 提示 (可选)                                        │
└─────────────────────────────────────────────────────────────────────────┘
```

### 4.2 溢出检测

**文件**: `packages/opencode/src/session/overflow.ts`

#### 4.2.1 可用上下文计算

```typescript
// 计算可用上下文 (usable context)
function usable(input: { cfg: Config.Info; model: Provider.Model }) {
  const context = input.model.limit.context
  if (context === 0) return 0

  // 保留缓冲: 20k 或 maxOutputTokens 的较小值
  const reserved = input.cfg.compaction?.reserved ?? 
    Math.min(COMPACTION_BUFFER, ProviderTransform.maxOutputTokens(input.model))
  
  // 如果模型有 input 限制，使用 input 限制; 否则使用总 context
  return input.model.limit.input
    ? Math.max(0, input.model.limit.input - reserved)
    : Math.max(0, context - ProviderTransform.maxOutputTokens(input.model))
}
```

**参数说明**:
- `COMPACTION_BUFFER`: 20,000 tokens，压缩预留缓冲
- `reserved`: 可配置，默认为 min(20k, max_output_tokens)
- `usable`: 实际可用于对话的 tokens 数

#### 4.2.2 溢出检测逻辑

```typescript
function isOverflow(input: { 
  cfg: Config.Info
  tokens: MessageV2.Assistant["tokens"]
  model: Provider.Model 
}) {
  // 如果配置禁用了自动压缩，返回 false
  if (input.cfg.compaction?.auto === false) return false
  
  // 无上下文限制的模型不触发压缩
  if (input.model.limit.context === 0) return false

  // 计算总 token 数
  const count = input.tokens.total || 
    input.tokens.input + input.tokens.output + 
    input.tokens.cache.read + input.tokens.cache.write
  
  // 超过可用上下文时触发
  return count >= usable(input)
}
```

**触发条件**:
- `tokens.total >= usable()` 即触发压缩
- 支持自动触发和手动触发

### 4.3 保留策略 (Tail Preservation)

**文件**: `packages/opencode/src/session/compaction.ts:36-43, 137-142, 247-296`

#### 4.3.1 配置参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| tail_turns | 2 | 保留最近 N 轮对话 |
| preserve_recent_tokens | 2000-8000 | 保留 tokens 数 (默认 25%) |
| reserved | min(20k, maxOutput) | 压缩预留缓冲 |

#### 4.3.2 预算计算

```typescript
// 计算保留预算
function preserveRecentBudget(input: { cfg: Config.Info; model: Provider.Model }) {
  return (
    // 配置优先
    input.cfg.compaction?.preserve_recent_tokens ??
    // 默认: 25% 的可用上下文，最少 2000，最多 8000
    Math.min(MAX_PRESERVE_RECENT_TOKENS, 
      Math.max(MIN_PRESERVE_RECENT_TOKENS, 
        Math.floor(usable(input) * 0.25)
      )
    )
  )
}

// 常量
const DEFAULT_TAIL_TURNS = 2            // 默认保留最近 2 轮
const MIN_PRESERVE_RECENT_TOKENS = 2_000 // 最少保留 2000 tokens
const MAX_PRESERVE_RECENT_TOKENS = 8_000 // 最多保留 8000 tokens
```

#### 4.3.3 选择算法

```typescript
// 消息轮次定义
type Turn = {
  start: number    // 起始索引
  end: number      // 结束索引
  id: MessageID    // 用户消息 ID
}

type Tail = {
  start: number    // 保留起始位置
  id: MessageID    // tail 起始消息 ID
}

// 分割消息为轮次 (按用户消息分割)
function turns(messages: MessageV2.WithParts[]) {
  const result: Turn[] = []
  for (let i = 0; i < messages.length; i++) {
    const msg = messages[i]
    // 跳过非用户消息和压缩标记消息
    if (msg.info.role !== "user") continue
    if (msg.parts.some((part) => part.type === "compaction")) continue
    result.push({
      start: i,
      end: messages.length,
      id: msg.info.id,
    })
  }
  // 设置每轮的结束位置为下一轮的起始位置
  for (let i = 0; i < result.length - 1; i++) {
    result[i].end = result[i + 1].start
  }
  return result
}

// 选择保留消息
function select(input: {
  messages: MessageV2.WithParts[]
  cfg: Config.Info
  model: Provider.Model
}) {
  // 1. 获取保留轮数限制
  const limit = input.cfg.compaction?.tail_turns ?? DEFAULT_TAIL_TURNS
  if (limit <= 0) return { head: input.messages, tail_start_id: undefined }
  
  // 2. 获取保留预算
  const budget = preserveRecentBudget({ cfg: input.cfg, model: input.model })
  
  // 3. 获取所有轮次
  const all = turns(input.messages)
  if (!all.length) return { head: input.messages, tail_start_id: undefined }
  
  // 4. 取最近 N 轮
  const recent = all.slice(-limit)
  
  // 5. 估算每轮 token 数
  const sizes = yield* Effect.forEach(
    recent,
    (turn) => estimate({ messages: input.messages.slice(turn.start, turn.end), model: input.model }),
    { concurrency: 1 },
  )

  // 6. 从后往前选择，累积预算内保留
  let total = 0
  let keep: Tail | undefined
  for (let i = recent.length - 1; i >= 0; i--) {
    const turn = recent[i]!
    const size = sizes[i]
    if (total + size <= budget) {
      total += size
      keep = { start: turn.start, id: turn.id }
      continue
    }
    // 7. 当前轮超出预算，尝试分割
    const remaining = budget - total
    const split = yield* splitTurn({
      messages: input.messages,
      turn,
      model: input.model,
      budget: remaining,
      estimate,
    })
    if (split) keep = split
    else if (!keep) log.info("tail fallback", { budget, size, total })
    break
  }

  // 8. 返回 head 和 tail 边界
  if (!keep || keep.start === 0) return { head: input.messages, tail_start_id: undefined }
  return {
    head: input.messages.slice(0, keep.start),  // 被压缩的部分
    tail_start_id: keep.id,                      // 保留部分的起始 ID
  }
}
```

**算法说明**:
1. 将消息按用户输入分割为轮次 (Turn)
2. 保留最近 N 轮 (tail_turns 默认 2 轮)
3. 从最旧的保留轮开始，累加 token 数量
4. 超出预算时，尝试在该轮内寻找分割点
5. 返回 head (待压缩) 和 tail_start_id (保留起始)

### 4.4 摘要生成

#### 4.4.1 摘要模板

**文件**: `packages/opencode/src/session/compaction.ts:43-78`

```typescript
const SUMMARY_TEMPLATE = `
Output exactly the Markdown structure shown inside <template> and keep the section order unchanged. Do not include the <template> tags in your response.
<template>
## Goal
- [single-sentence task summary]

## Constraints & Preferences
- [user constraints, preferences, specs, or "(none)"]

## Progress
### Done
- [completed work or "(none)"]

### In Progress
- [current work or "(none)"]

### Blocked
- [blockers or "(none)"]

## Key Decisions
- [decision and why, or "(none)"]

## Next Steps
- [ordered next actions or "(none)"]

## Critical Context
- [important technical facts, errors, open questions, or "(none)"]

## Relevant Files
- [file or directory path: why it matters, or "(none)"]
</template>

Rules:
- Keep every section, even when empty.
- Use terse bullets, not prose paragraphs.
- Preserve exact file paths, commands, error strings, and identifiers when known.
- Do not mention the summary process or that context was compacted.`
```

#### 4.4.2 摘要构建

```typescript
function buildPrompt(input: { previousSummary?: string; context: string[] }) {
  // 如果有之前的摘要，使用增量更新模式
  const anchor = input.previousSummary
    ? [
        "Update the anchored summary below using the conversation history above.",
        "Preserve still-true details, remove stale details, and merge in the new facts.",
        "<previous-summary>",
        input.previousSummary,
        "</previous-summary>",
      ].join("\n")
    : "Create a new anchored summary from the conversation history above."
  
  return [anchor, SUMMARY_TEMPLATE, ...input.context].join("\n\n")
}
```

**特点**:
- 支持增量摘要 (previousSummary)
- 使用 Markdown 模板结构化输出
- 不暴露压缩过程给模型

### 4.5 工具输出裁剪 (Pruning)

**文件**: `packages/opencode/src/session/compaction.ts:298-344`

#### 4.5.1 配置参数

| 参数 | 值 | 说明 |
|------|-----|------|
| PRUNE_MINIMUM | 20,000 | 最小裁剪量 |
| PRUNE_PROTECT | 40,000 | 保护阈值 |
| PRUNE_PROTECTED_TOOLS | [skill] | 保护工具列表 |
| TOOL_OUTPUT_MAX_CHARS | 2,000 | 工具输出最大字符数 |

#### 4.5.2 裁剪算法

```typescript
function prune(input: { sessionID: SessionID }) {
  const cfg = yield* config.get()
  if (!cfg.compaction?.prune) return  // 配置禁用则跳过
  log.info("pruning")

  const msgs = yield* session.messages({ sessionID: input.sessionID })
  if (!msgs) return

  let total = 0
  let pruned = 0
  const toPrune: MessageV2.ToolPart[] = []
  let turns = 0

  // 从后往前遍历消息
  loop: for (let msgIndex = msgs.length - 1; msgIndex >= 0; msgIndex--) {
    const msg = msgs[msgIndex]
    
    // 统计轮数
    if (msg.info.role === "user") turns++
    
    // 保留最近 2 轮对话
    if (turns < 2) continue
    
    // 遇到摘要消息停止
    if (msg.info.role === "assistant" && msg.info.summary) break loop
    
    // 遍历消息的所有 parts
    for (let partIndex = msg.parts.length - 1; partIndex >= 0; partIndex--) {
      const part = msg.parts[partIndex]
      
      // 只处理 tool 类型
      if (part.type !== "tool") continue
      
      // 只处理已完成状态
      if (part.state.status !== "completed") continue
      
      // 保护指定工具
      if (PRUNE_PROTECTED_TOOLS.includes(part.tool)) continue
      
      // 跳过已裁剪的
      if (part.state.time.compacted) break loop
      
      // 累加 token 估计
      const estimate = Token.estimate(part.state.output)
      total += estimate
      
      // 未超过保护阈值则继续
      if (total <= PRUNE_PROTECT) continue
      
      // 超过阈值，标记待裁剪
      pruned += estimate
      toPrune.push(part)
    }
  }

  log.info("found", { pruned, total })
  
  // 执行裁剪
  if (pruned > PRUNE_MINIMUM) {
    for (const part of toPrune) {
      if (part.state.status === "completed") {
        part.state.time.compacted = Date.now()  // 标记裁剪时间
        yield* session.updatePart(part)
      }
    }
    log.info("pruned", { count: toPrune.length })
  }
}
```

**裁剪逻辑**:
1. 从最旧的消息开始向前扫描
2. 保留最近 2 轮对话 (turns < 2 跳过)
3. 遇到 summary 消息停止
4. 对 tool 类型的 completed 状态输出进行裁剪
5. 累积超过 PRUNE_PROTECT (40k) 后标记待裁剪
6. 总裁剪量超过 PRUNE_MINIMUM (20k) 时执行裁剪

### 4.6 配置项

**文件**: `packages/opencode/src/config/config.ts:227-246`

```typescript
compaction: Schema.Struct({
  // 自动压缩开关
  auto: Schema.optional(Schema.Boolean).annotate({
    description: "Enable automatic compaction when context is full (default: true)",
  }),
  
  // 工具输出裁剪开关
  prune: Schema.optional(Schema.Boolean).annotate({
    description: "Enable pruning of old tool outputs (default: true)",
  }),
  
  // 保留轮数
  tail_turns: Schema.optional(NonNegativeInt).annotate({
    description: "Number of recent user turns to keep verbatim during compaction (default: 2)",
  }),
  
  // 保留 tokens 数
  preserve_recent_tokens: Schema.optional(NonNegativeInt).annotate({
    description: "Maximum number of tokens from recent turns to preserve verbatim after compaction",
  }),
  
  // 预留缓冲
  reserved: Schema.optional(NonNegativeInt).annotate({
    description: "Token buffer for compaction. Leaves enough window to avoid overflow during compaction.",
  }),
})
```

### 4.7 插件扩展点

**文件**: `packages/opencode/src/session/compaction.ts:399-405`

```typescript
// 允许插件注入上下文或替换压缩 prompt
const compacting = yield* plugin.trigger(
  "experimental.session.compacting",
  { sessionID: input.sessionID },
  { context: [], prompt: undefined },
)
const nextPrompt = compacting.prompt ?? buildPrompt({ previousSummary, context: compacting.context })

// 允许插件转换消息
yield* plugin.trigger("experimental.chat.messages.transform", {}, { messages: msgs })

// 自动继续插件
if ((yield* plugin.trigger("experimental.compaction.autocontinue", {...})).enabled) {
  // 发送 Continue 提示
}
```

**扩展点**:
- `experimental.session.compacting`: 注入额外上下文或替换压缩 prompt
- `experimental.chat.messages.transform`: 在压缩前转换消息
- `experimental.compaction.autocontinue`: 控制压缩后是否自动继续

---

## 5. 关键文件索引

| 文件 | 说明 |
|------|------|
| `src/storage/db.ts` | 数据库连接配置 |
| `src/session/session.sql.ts` | 5 个表结构定义 |
| `src/session/overflow.ts` | 溢出检测逻辑 (26 行) |
| `src/session/compaction.ts` | 压缩机制 (652 行) |
| `src/config/config.ts` | 配置项定义 |
| `src/v2/session-message-updater.ts` | 内存状态更新 |

---

## 6. 总结

- **SQLite 存储**: WAL 模式，三层结构 (Session → Message → Part)
- **事件驱动**: 内存状态实时更新
- **自动压缩**: 基于 tokens.total >= usable() 触发
- **智能保留**: tail_turns (默认 2 轮) + preserve_recent_tokens (默认 25%)
- **工具裁剪**: 保护 skill 等工具，释放旧输出空间
- **插件扩展**: 支持压缩上下文注入、消息转换、自动继续控制