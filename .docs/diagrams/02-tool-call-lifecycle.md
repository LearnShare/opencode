# Tool Call 生命周期

> 本文档详细描述 OpenCode 中 Tool Call 的完整生命周期，从 LLM 生成调用到结果返回的整个过程。

---

## 1. 整体流程图

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                                    LLM 生成 Tool Call                                   │
│                           (streamText 返回 tool-call 事件)                              │
└─────────────────────────────────────────────────────────────────────────────────────────┘
                                             │
                                             ▼
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                           SessionProcessor.handleEvent()                                │
│                              处理 tool-call 相关事件                                    │
└─────────────────────────────────────────────────────────────────────────────────────────┘
                                             │
                     ┌───────────────────────┼───────────────────────┐
                     │                       │                       │
                     ▼                       ▼                       ▼
        ┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐
        │ tool-input-start │     │    tool-call     │     │ tool-input-end  │
        │   (建立追踪)     │     │   (执行工具)     │     │   (完成输入)     │
        └──────────────────┘     └──────────────────┘     └──────────────────┘
                     │                       │                       │
                     │                       ▼                       │
                     │              ┌──────────────────┐              │
                     │              │  工具权限检查    │              │
                     │              │ Permission.eval │              │
                     │              └──────────────────┘              │
                     │                       │                       │
                     │          ┌────────────┴────────────┐          │
                     │          ▼                         ▼          │
                     │   ┌──────────────┐        ┌──────────────┐  │
                     │   │   allow      │        │    deny      │  │
                     │   │   执行工具    │        │  返回错误    │  │
                     │   └──────────────┘        └──────────────┘  │
                     │                       │                       │
                     │                       ▼                       │
                     │              ┌──────────────────┐              │
                     │              │  Tool.execute() │              │
                     │              │  (AI SDK 调用)  │              │
                     │              └──────────────────┘              │
                     │                       │                       │
                     │          ┌────────────┴────────────┐          │
                     │          ▼                         ▼          │
                     │   ┌──────────────┐        ┌──────────────┐  │
                     │   │   成功       │        │    失败      │  │
                     │   │ tool-result  │        │ tool-error   │  │
                     │   └──────────────┘        └──────────────┘  │
                     │           │                       │           │
                     └───────────┼───────────────────────┼───────────┘
                                 │                       │
                                 ▼                       ▼
                    ┌────────────────────┐    ┌────────────────────┐
                    │   完成 Tool Call   │    │   标记 Tool Call    │
                    │   completeToolCall │    │   failToolCall     │
                    └────────────────────┘    └────────────────────┘
                                 │                       │
                                 └───────────┬───────────┘
                                             │
                                             ▼
                    ┌────────────────────────────────────────────────────────────┐
                    │                    结果返回 LLM                           │
                    │         (tool-result 作为消息发送回 LLM 继续对话)           │
                    └────────────────────────────────────────────────────────────┘
                                             │
                                             ▼
                    ┌────────────────────────────────────────────────────────────┐
                    │                    循环继续或结束                         │
                    └────────────────────────────────────────────────────────────┘
```

---

## 2. 详细状态机

### 2.1 Tool 状态定义

```typescript
// message-v2.ts
type ToolState = 
  | { status: "pending" }      // 待调用
  | { status: "running"; input: any; time: { start: number } }
  | { status: "completed"; input: any; output: string; time: { start: number; end: number } }
  | { status: "error"; input: any; error: string; time: { start: number; end: number } }
```

### 2.2 状态转换图

```
┌──────────┐  tool-call   ┌─────────┐  执行完成   ┌────────────┐
│ pending  │ ──────────► │ running │ ──────────► │ completed  │
└──────────┘             └─────────┘             └────────────┘
       │                      │
       │                      │ 执行失败
       ▼                      ▼
┌────────────┐          ┌──────────┐
│   error    │ ◄────────│  error   │
└────────────┘          └──────────┘
```

---

## 3. 事件序列

### 3.1 成功执行序列

```
LLM                    SessionProcessor              Tool                  Session
 │                          │                          │                     │
 │ ──────────────────────► │                          │                     │
 │ tool-input-start        │                          │                     │
 │ (id: "call_xxx")        │                          │                     │
 │                          │ ──────────────────────► │                     │
 │                          │                          │                     │
 │                          │     tool-call            │                     │
 │                          │     (toolName, input)    │                     │
 │                          │                          │                     │
 │                          │   ┌─────────────────┐    │                     │
 │                          │   │ Permission.eval │    │                     │
 │                          │   └─────────────────┘    │                     │
 │                          │          │               │                     │
 │                          │          ▼               │                     │
 │                          │   ┌─────────────────┐    │                     │
 │                          │   │ Tool.execute()  │    │                     │
 │                          │   └─────────────────┘    │                     │
 │                          │          │               │                     │
 │                          │          ▼               │                     │
 │                          │   执行成功               │                     │
 │                          │                          │                     │
 │                          │ ◄────────────────────── │                     │
 │                          │    tool-result          │                     │
 │                          │    {output, title,      │                     │
 │                          │     metadata, files}    │                     │
 │                          │                          │                     │
 │                          │ ─────────────────────► │                     │
 │                          │   completeToolCall()   │                     │
 │                          │                          │                     │
 │                          │                      ◄─┤                     │
 │                          │   updatePart()         │                     │
 │                          │                          │                     │
 │ ◄────────────────────── │                          │                     │
 │ tool-result (as msg)   │                          │                     │
 │ (继续对话循环)          │                          │                     │
 │                          │                          │                     │
```

### 3.2 错误执行序列

```
LLM                    SessionProcessor              Tool                  Session
 │                          │                          │                     │
 │ ──────────────────────► │                          │                     │
 │ tool-input-start        │                          │                     │
 │                          │ ──────────────────────► │                     │
 │                          │     tool-call            │                     │
 │                          │                          │                     │
 │                          │   ┌─────────────────┐    │                     │
 │                          │   │ Permission.eval │    │                     │
 │                          │   └─────────────────┘    │                     │
 │                          │          │               │                     │
 │                          │          ▼               │                     │
 │                          │   ┌─────────────────┐    │                     │
 │                          │   │ Tool.execute() │    │                     │
 │                          │   └─────────────────┘    │                     │
 │                          │          │               │                     │
 │                          │          ▼               │                     │
 │                          │   执行失败 (throw)       │                     │
 │                          │                          │                     │
 │                          │ ◄────────────────────── │                     │
 │                          │    tool-error           │                     │
 │                          │    {error: ...}         │                     │
 │                          │                          │                     │
 │                          │ ◄───────────────────── │                     │
 │                          │   failToolCall()        │                     │
 │                          │                          │                     │
 │                          │ ◄───────────────────── │                     │
 │                          │   updatePart(status:    │                     │
 │                          │          "error")       │                     │
 │                          │                          │                     │
 │ ◄────────────────────── │                          │                     │
 │ tool-result (error)    │                          │                     │
 │ (继续对话循环)          │                          │                     │
```

---

## 4. 核心代码解析

### 4.1 Processor 处理 Tool Call (processor.ts:319-376)

```typescript
case "tool-call": {
  // 读取已存在的 tool call
  const toolCall = yield* readToolCall(value.toolCallId)
  
  // 发布事件
  EventV2.run(SessionEvent.Tool.Called.Sync, {
    sessionID: ctx.sessionID,
    callID: value.toolCallId,
    tool: value.toolName,
    input: value.input,
    ...
  })
  
  // 更新 tool part 状态为 running
  yield* updateToolCall(value.toolCallId, (match) => ({
    ...match,
    tool: value.toolName,
    state: {
      ...match.state,
      status: "running",
      input: value.input,
      time: { start: Date.now() },
    },
  }))
  
  // Doom Loop 检测 (连续 3 次相同工具调用)
  const recentParts = parts.slice(-DOOM_LOOP_THRESHOLD)
  if (isDoomLoop(recentParts, value)) {
    yield* permission.ask({ permission: "doom_loop", ... })
  }
  return
}
```

### 4.2 Tool 执行结果处理 (processor.ts:378-403)

```typescript
case "tool-result": {
  const toolCall = yield* readToolCall(value.toolCallId)
  
  // 发布成功事件
  EventV2.run(SessionEvent.Tool.Success.Sync, {
    sessionID: ctx.sessionID,
    callID: value.toolCallId,
    structured: value.output.metadata,
    content: [...],
  })
  
  // 完成 tool call
  yield* completeToolCall(value.toolCallId, {
    title: value.output.title,
    metadata: value.output.metadata,
    output: value.output.output,
    attachments: value.output.attachments,
  })
  return
}
```

### 4.3 Tool 错误处理 (processor.ts:406-422)

```typescript
case "tool-error": {
  const toolCall = yield* readToolCall(value.toolCallId)
  
  // 发布失败事件
  EventV2.run(SessionEvent.Tool.Failed.Sync, {
    sessionID: ctx.sessionID,
    callID: value.toolCallId,
    error: { type: "unknown", message: errorMessage(value.error) },
    ...
  })
  
  // 标记为失败
  yield* failToolCall(value.toolCallId, value.error)
  return
}
```

### 4.4 completeToolCall (processor.ts:179-199)

```typescript
const completeToolCall = Effect.fn("SessionProcessor.completeToolCall")(
  function* (
    toolCallID: string,
    output: { title: string; metadata: Record<string, any>; output: string; attachments?: FilePart[] }
  ) {
    const match = yield* readToolCall(toolCallID)
    if (!match || match.part.state.status !== "running") return
    
    // 更新 part 状态为 completed
    yield* session.updatePart({
      ...match.part,
      state: {
        status: "completed",
        output: output.output,
        title: output.title,
        metadata: output.metadata,
        time: { start: match.part.state.time.start, end: Date.now() },
      },
      attachments: output.attachments,
    })
    
    // 标记完成
    yield* settleToolCall(toolCallID)
  }
)
```

---

## 5. 权限检查

### 5.1 权限评估 (permission.ts)

```typescript
// 权限类型
type Ruleset = {
  allow?: string[]    // 允许的模式
  deny?: string[]     // 拒绝的模式
  ask?: string[]     // 询问用户的模式
}

// 评估结果
type Result = "allow" | "deny" | "ask"
```

### 5.2 权限检查流程

```
┌────────────────────────────────────────────────┐
│           Permission.evaluate()               │
├────────────────────────────────────────────────┤
│ 1. 检查 deny 规则 → deny                        │
│ 2. 检查 allow 规则 → allow                      │
│ 3. 检查 ask 规则 → ask                          │
│ 4. 默认 allow                                  │
└────────────────────────────────────────────────┘
```

### 5.3 Doom Loop 检测 (processor.ts:350-375)

```typescript
// 连续 3 次相同工具调用相同参数
const recentParts = parts.slice(-DOOM_LOOP_THRESHOLD)  // 3

if (recentParts.every(part => 
  part.type === "tool" &&
  part.tool === value.toolName &&
  part.state.status !== "pending" &&
  JSON.stringify(part.state.input) === JSON.stringify(value.input)
)) {
  // 触发权限询问
  yield* permission.ask({
    permission: "doom_loop",
    patterns: [value.toolName],
    ...
  })
}
```

---

## 6. 数据模型

### 6.1 ToolPart 结构

```typescript
// message-v2.ts
interface ToolPart {
  id: PartID
  messageID: MessageID
  sessionID: SessionID
  type: "tool"
  tool: string
  callID: string
  state: ToolState
  metadata?: {
    providerExecuted?: boolean  // Provider 内置执行
    interrupted?: boolean       // 中断
  }
  attachments?: FilePart[]      // 文件附件
}
```

### 6.2 Tool 定义

```typescript
// tool/tool.ts
interface ToolDef {
  name: string
  description: string
  parameters: JSONSchema7
  execute: (input: any, metadata: ToolExecutionMetadata) => Promise<ToolResult>
}

// Tool 执行元数据
interface ToolExecutionMetadata {
  toolCallId: string
  messages: Message[]
  abortSignal: AbortSignal
}
```

---

## 7. 关键文件索引

| 文件 | 说明 |
|------|------|
| `packages/opencode/src/session/processor.ts` | Tool Call 事件处理 |
| `packages/opencode/src/session/message-v2.ts` | ToolPart 数据模型 |
| `packages/opencode/src/tool/registry.ts` | 工具注册表 |
| `packages/opencode/src/permission/permission.ts` | 权限评估 |
| `packages/opencode/src/session/llm.ts` | LLM 服务 (工具执行) |

---

## 8. 总结

Tool Call 生命周期:

1. **LLM 生成**: LLM 返回 tool_call 事件
2. **事件接收**: `handleEvent("tool-call")` 处理
3. **状态更新**: 更新 part 状态为 running
4. **权限检查**: `Permission.evaluate()` 评估
5. **执行工具**: AI SDK 调用 tool.execute()
6. **结果处理**: 成功 → `completeToolCall()`, 失败 → `failToolCall()`
7. **消息发送**: 结果作为消息发送回 LLM
8. **循环继续**: 等待 LLM 下一轮响应