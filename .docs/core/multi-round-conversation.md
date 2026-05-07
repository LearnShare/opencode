# 多轮对话上下文保持机制

> 研究时间: ~4 小时
> 状态: ✅ 已完成
> 关联 Phase: 核心机制

---

## 1. 概述

在 OpenCode 的 Session 中，多轮对话的上下文保持是一个核心问题。本文档深入分析系统如何确保:
1. **上下文完整**: 历史消息正确传递给 LLM
2. **任务持续**: 在多轮交互中保持任务状态
3. **上下文压缩**: 超出限制时智能压缩

---

## 2. 核心机制

### 2.1 消息加载机制

**入口**: `SessionPrompt.runLoop()` (prompt.ts:1400-1627)

```typescript
const runLoop: (sessionID: SessionID) => Effect.Effect<MessageV2.WithParts> = 
  Effect.fn("SessionPrompt.run")(function* (sessionID: SessionID) {
    while (true) {
      // 关键: 每次循环都重新加载消息
      let msgs = yield* MessageV2.filterCompactedEffect(sessionID)
      
      // ... 处理逻辑
    }
  })
```

**关键点**: 每次循环都从数据库加载完整消息历史，确保数据最新。

---

## 3. 消息流处理

### 3.1 消息加载 (filterCompactedEffect)

**文件**: `message-v2.ts:1136-1138`

```typescript
export const filterCompactedEffect = Effect.fnUntraced(function* (sessionID: SessionID) {
  return filterCompacted(stream(sessionID))
})

// stream 函数 - 按时间倒序分页加载
export function* stream(sessionID: SessionID) {
  const size = 50
  let before: string | undefined
  while (true) {
    const next = page({ sessionID, limit: size, before })
    if (next.items.length === 0) break
    for (let i = next.items.length - 1; i >= 0; i--) {
      yield next.items[i]  // 倒序 yield，保持时间顺序
    }
    if (!next.more || !next.cursor) break
    before = next.cursor
  }
}
```

**流程**:
```
Database (按时间倒序)
    ↓
分页加载 (每页 50 条)
    ↓
倒序遍历 (保证时间顺序)
    ↓
filterCompacted (过滤压缩消息)
```

### 3.2 压缩过滤 (filterCompacted)

**文件**: `message-v2.ts:1083-1134`

```typescript
export function filterCompacted(msgs: Iterable<WithParts>) {
  const result = [] as WithParts[]
  const completed = new Set<string>()
  let retain: MessageID | undefined

  for (const msg of msgs) {
    result.push(msg)

    // 如果已确定保留边界，跳过中间消息
    if (retain) {
      if (msg.info.id === retain) break
      continue
    }

    // 找到包含压缩摘要的用户消息
    if (msg.info.role === "user" && completed.has(msg.info.id)) {
      const part = msg.parts.find((item): item is CompactionPart => item.type === "compaction")
      if (!part) continue
      if (!part.tail_start_id) break  // 无保留起始，跳过
      retain = part.tail_start_id    // 设置保留边界
      if (msg.info.id === retain) break
      continue
    }

    // 遇到已完成摘要，标记对应用户消息完成
    if (msg.info.role === "assistant" && msg.info.summary && msg.info.finish && !msg.info.error) {
      completed.add(msg.info.parentID)
    }
  }

  // 重新排序为时间正序
  result.reverse()

  // 处理压缩后的消息拼接
  // ... 返回完整消息链
}
```

---

## 4. 消息链构建

### 4.1 转换为 LLM 消息格式 (toModelMessagesEffect)

**文件**: `message-v2.ts:729-999`

```typescript
export const toModelMessagesEffect = Effect.fnUntraced(function* (
  input: WithParts[],
  model: Provider.Model,
  options?: { stripMedia?: boolean; toolOutputMaxChars?: number },
) {
  const result: UIMessage[] = []

  for (const msg of input) {
    if (msg.parts.length === 0) continue

    // 用户消息处理
    if (msg.info.role === "user") {
      const userMessage: UIMessage = { id: msg.info.id, role: "user", parts: [] }
      result.push(userMessage)
      for (const part of msg.parts) {
        if (part.type === "text" && !part.ignored)
          userMessage.parts.push({ type: "text", text: part.text })
        if (part.type === "file")
          userMessage.parts.push({ type: "file", url: part.url, mediaType: part.mime })
        if (part.type === "compaction")  // 压缩提示
          userMessage.parts.push({ type: "text", text: "What did we do so far?" })
      }
    }

    // 助手消息处理
    if (msg.info.role === "assistant") {
      const assistantMessage: UIMessage = { id: msg.info.id, role: "assistant", parts: [] }
      result.push(assistantMessage)
      for (const part of msg.parts) {
        if (part.type === "text" && !part.ignored)
          assistantMessage.parts.push({ type: "text", text: part.text })

        // 工具调用
        if (part.type === "tool") {
          if (part.state.status === "running") {
            // 运行中的工具 - 转换为 tool-call
            assistantMessage.parts.push({
              type: "tool-call",
              toolCallId: part.callID,
              toolName: part.tool,
              input: part.state.input,
            })
          }
          if (part.state.status === "completed") {
            // 已完成工具 - 转换为 tool-result
            // 注意: 不再追加到 assistant 消息，而是等待下一轮作为独立 user 消息
          }
        }
      }
    }

    // Tool Result 处理 - 作为独立的 user 消息
    for (const msg of input) {
      for (const part of msg.parts) {
        if (msg.info.role === "assistant" && part.type === "tool" && part.state.status === "completed") {
          result.push({
            role: "tool",
            content: part.state.output,
            toolCallId: part.callID,
          })
        }
      }
    }
  }

  return result
})
```

### 4.2 消息格式转换流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        MessageV2.WithParts[]                                │
│  [user msg + parts] → [assistant msg + parts] → [tool result parts]        │
└─────────────────────────────────────────────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                        toModelMessagesEffect()                               │
└─────────────────────────────────────────────────────────────────────────────┘
                                     │
         ┌───────────────────────────┼───────────────────────────┐
         │                           │                           │
         ▼                           ▼                           ▼
┌──────────────────┐    ┌──────────────────────┐    ┌──────────────────────┐
│ User Message     │    │ Assistant Message    │    │ Tool Result Message │
│                  │    │                      │    │                     │
│ {                │    │ {                    │    │ {                   │
│   role: "user",  │    │   role: "assistant", │    │   role: "tool",    │
│   parts: [       │    │   parts: [           │    │   content: "...",   │
│     {type:"text"│    │     {type:"text",    │    │   toolCallId: ".."  │
│     },           │    │     },               │    │ }                   │
│     {type:"file"│    │     {type:"tool-call"│    │                     │
│     }            │    │     }                │    │                     │
│   ]              │    │   ]                   │    │                     │
│ }                │    │ }                    │    │                     │
└──────────────────┘    └──────────────────────┘    └──────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                        AI SDK ModelMessage[]                                 │
│  格式: [{ role: "user", content: ... }, { role: "assistant", content: ... }]│
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 5. 任务持续机制

### 5.1 消息父子关系

**文件**: `message-v2.ts`

```typescript
// 用户消息结构
interface User {
  id: MessageID
  parentID?: MessageID    // 指向上一条消息
  role: "user"
  agent: string
  model: ModelRef
  parts: Part[]
}

// 助手消息结构
interface Assistant {
  id: MessageID
  parentID: MessageID     // 指向用户消息
  role: "assistant"
  agent: string
  finish?: string         // "stop" | "tool-calls" | "unknown"
  error?: object
  tokens: Usage
  // ...
}
```

**关键**: `parentID` 建立消息链，确保上下文连续。

### 5.2 循环退出条件

**文件**: `prompt.ts:1440-1448`

```typescript
// 检查是否有未完成的 tool calls
const hasToolCalls =
  lastAssistantMsg?.parts.some((part) => part.type === "tool" && !part.metadata?.providerExecuted) ?? false

// 退出条件: 完成且无 tool calls
if (
  lastAssistant?.finish &&
  !["tool-calls"].includes(lastAssistant.finish) &&
  !hasToolCalls &&
  lastUser.id < lastAssistant.id
) {
  yield* slog.info("exiting loop")
  break  // 退出循环
}
```

**逻辑**:
- `finish === "stop"`: LLM 正常结束
- `hasToolCalls === false`: 无待处理工具调用
- `lastUser.id < lastAssistant.id`: 用户消息在助手消息之前 (有效对)

### 5.3 步骤追踪

**文件**: `processor.ts:428-451`

```typescript
case "start-step":
  if (!ctx.snapshot) ctx.snapshot = yield* snapshot.track()

  // 发布步骤开始事件
  EventV2.run(SessionEvent.Step.Started.Sync, {
    sessionID: ctx.sessionID,
    agent: input.assistantMessage.agent,
    model: {...},
    snapshot: ctx.snapshot,
    timestamp: DateTime.makeUnsafe(Date.now()),
  })

  // 记录步骤开始
  yield* session.updatePart({
    id: PartID.ascending(),
    messageID: ctx.assistantMessage.id,
    sessionID: ctx.sessionID,
    snapshot: ctx.snapshot,
    type: "step-start",
  })
  return
```

---

## 6. 上下文压缩机制

### 6.1 触发条件

**文件**: `session/overflow.ts`

```typescript
function isOverflow(input: { 
  cfg: Config.Info
  tokens: MessageV2.Assistant["tokens"]
  model: Provider.Model 
}) {
  // 禁用自动压缩
  if (input.cfg.compaction?.auto === false) return false
  
  // 无上下文限制的模型不触发
  if (input.model.limit.context === 0) return false

  // 计算总 token 数
  const count = input.tokens.total || 
    input.tokens.input + input.tokens.output + 
    input.tokens.cache.read + input.tokens.cache.write
  
  // 超过可用上下文
  return count >= usable(input)
}
```

### 6.2 压缩流程

**文件**: `session/compaction.ts`

```typescript
// 压缩流程
const compact = Effect.fn("SessionCompaction.create")(function* (input: CreateInput) {
  // 1. 添加压缩任务到消息
  yield* session.updateMessage({
    ...userMessage,
    parts: [...userMessage.parts, {
      type: "compaction",
      auto: input.auto,
      overflow: input.overflow,
    }]
  })

  // 2. runLoop 会处理压缩任务
  // 3. 生成摘要
  // 4. 更新消息链
})
```

### 6.3 保留策略 (Tail Preservation)

```typescript
// 配置参数
const DEFAULT_TAIL_TURNS = 2            // 保留最近 2 轮对话
const MIN_PRESERVE_RECENT_TOKENS = 2_000
const MAX_PRESERVE_RECENT_TOKENS = 8_000

// 算法:
// 1. 按用户消息分割为轮次 (Turn)
// 2. 保留最近 N 轮 (tail_turns)
// 3. 在保留轮内按 token 预算选择
// 4. 生成摘要替换被压缩的消息
```

---

## 7. 数据结构

### 7.1 数据库表结构

**文件**: `session.sql.ts`

```typescript
// SessionMessage 表 (V2)
const SessionMessageTable = sqliteTable("session_message", {
  id: text("id").primaryKey(),
  session_id: text("session_id").notNull(),
  type: text("type").notNull(),  // user/assistant/shell/compaction/...
  time_created: integer().notNull(),
  time_updated: integer().notNull(),
  data: json("data"),            // 消息内容
})
```

### 7.2 Part 类型

```typescript
type Part = 
  | TextPart       // 文本内容
  | ToolPart       // 工具调用/结果
  | ReasoningPart  // 推理过程
  | FilePart       // 文件附件
  | CompactionPart // 压缩标记
  | SubtaskPart    // 子任务
  | StepStartPart  // 步骤开始
  | StepFinishPart // 步骤结束
  | PatchPart      // 代码变更
```

---

## 8. 完整流程示例

### 8.1 第一轮

```
1. 用户输入: "帮我写一个 hello world 程序"

2. runLoop 加载消息:
   [] (空)

3. 构建 StreamInput:
   - user: 用户消息
   - messages: [user]
   - tools: [Read, Write, Shell, ...]

4. LLM 返回:
   - text: "我来帮你..."
   - tool-call: Write(filePath="hello.py", content="...")

5. tool 执行后:
   - Message 链: [user(msg1)] → [assistant(msg2) + tool(part)]
   - 继续循环

6. 第二轮:
   - messages: [user(msg1), assistant(msg2), tool-result(msg3)]
   - LLM 返回: "已创建文件，运行一下"
   - tool-call: Shell(command="python hello.py")
```

### 8.2 多轮后上下文溢出

```
1. Token 超限检测:
   isOverflow({ tokens: { total: 120000 }, model: { limit: { context: 128000 } } })
   → true

2. 触发压缩:
   - 添加 compaction 任务
   - 保留最近 2 轮 (tail_turns: 2)
   - 调用压缩模型生成摘要

3. 消息链更新:
   [user(msg1)] ──压缩──→ [assistant(summary)]
                              ↑
                     [user(msgN-1), assistant(msgN), tool-result]

4. 继续对话时:
   - 加载: [user(summary), user(msgN-1), ...]
   - 摘要作为上下文起点
```

---

## 9. 关键文件索引

| 文件 | 说明 |
|------|------|
| `src/session/prompt.ts` | 主循环逻辑，消息加载 |
| `src/session/message-v2.ts` | 消息数据模型，转换函数 |
| `src/session/processor.ts` | 事件处理，步骤追踪 |
| `src/session/compaction.ts` | 上下文压缩 |
| `src/session/overflow.ts` | 溢出检测 |
| `src/session/run-state.ts` | 运行状态管理 |

---

## 10. 总结

### 上下文保持机制

1. **消息加载**: 每次循环从数据库加载完整消息链 (`filterCompactedEffect`)
2. **格式转换**: MessageV2 → ModelMessage (`toModelMessagesEffect`)
3. **父子关系**: parentID 建立消息链
4. **退出条件**: finish + 无 tool calls + 有效消息对

### 任务持续机制

1. **循环控制**: while(true) + 退出条件检查
2. **步骤追踪**: start-step / finish-step 事件
3. **状态保存**: 消息状态实时写入数据库

### 压缩机制

1. **触发**: tokens.total >= usable(context)
2. **保留**: tail_turns (默认 2 轮) + 25% token 预算
3. **摘要**: 增量更新，保留关键信息