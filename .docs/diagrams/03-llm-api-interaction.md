# LLM API 交互与数据结构

> 本文档详细描述在完整 Session 中，OpenCode 与 LLM API 之间的交互以及每次传输的完整数据结构。

---

## 1. 交互流程概览

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                                   用户输入                                              │
│                              (run "修复 bug")                                          │
└─────────────────────────────────────────────────────────────────────────────────────────┘
                                             │
                                             ▼
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                               SessionPrompt.loop()                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────────┐   │
│  │ 1. 加载消息历史 (filterCompactedEffect)                                        │   │
│  │ 2. 构建 System Prompt (sys.environment, instruction.system, sys.skills)     │   │
│  │ 3. 转换消息格式 (MessageV2.toModelMessagesEffect)                            │   │
│  │ 4. 构建 Tool 定义 (resolveTools)                                              │   │
│  └─────────────────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────────────────┘
                                             │
                                             ▼
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                           LLM.Service.stream() - 第一次调用                           │
│  ┌─────────────────────────────────────────────────────────────────────────────────┐   │
│  │ StreamInput 发送至 LLM API                                                   │   │
│  └─────────────────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────────────────┘
                                             │
                                             ▼
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                                 LLM API 返回流                                          │
│                    (AI SDK streamText 生成的事件流)                                    │
└─────────────────────────────────────────────────────────────────────────────────────────┘
                                             │
                                             ▼
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                         SessionProcessor.handleEvent() 处理                           │
│  ┌─────────────────────────────────────────────────────────────────────────────────┐   │
│  │ 事件处理: text-delta, reasoning-delta, tool-call, tool-result, finish-step   │   │
│  └─────────────────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────────────────┘
                                             │
                   ┌─────────────────────────┼─────────────────────────┐
                   │                         │                         │
                   ▼                         ▼                         ▼
          ┌───────────────┐         ┌───────────────┐         ┌───────────────┐
          │  Tool Call    │         │   文本输出    │         │   推理输出    │
          │  工具执行后    │         │  返回给 LLM   │         │  (reasoning)  │
          │  再次调用 LLM  │         │   循环结束    │         │  继续循环     │
          └───────────────┘         └───────────────┘         └───────────────┘
```

---

## 2. 第一次 LLM 调用 - StreamInput 结构

### 2.1 StreamInput 完整结构

```typescript
// llm.ts:36-49
type StreamInput = {
  // 用户消息
  user: MessageV2.User
  
  // 会话信息
  sessionID: string
  parentSessionID?: string
  
  // 模型信息
  model: Provider.Model
  agent: Agent.Info
  
  // 权限
  permission?: Permission.Ruleset
  
  // System Prompt
  system: string[]
  
  // 对话历史消息
  messages: ModelMessage[]
  
  // 是否使用小模型
  small?: boolean
  
  // Tool 定义
  tools: Record<string, Tool>
  
  // 重试次数
  retries?: number
  
  // 工具选择模式
  toolChoice?: "auto" | "required" | "none"
}
```

### 2.2 user (MessageV2.User)

```typescript
interface MessageV2.User {
  id: MessageID
  parentID?: MessageID
  role: "user"
  agent: string
  model: {
    providerID: ProviderID
    modelID: ModelID
    variant?: string
  }
  parts: Array<TextPart | FilePart | AgentPart | SubtaskPart>
  system?: string
  format?: { type: "text" } | { type: "json_schema"; schema: JSONSchema7 }
}
```

### 2.3 messages (对话历史) - ModelMessage[]

```typescript
// AI SDK ModelMessage 类型
type ModelMessage = 
  | { role: "user"; content: ModelContent }
  | { role: "assistant"; content: ModelContent }
  | { role: "tool"; content: string; toolCallId: string }

type ModelContent = 
  | string
  | Array<{ type: "text"; text: string } | { type: "tool-result"; toolCallId: string; result: string }>
```

**示例**:
```typescript
[
  // System 消息 (工具定义)
  {
    role: "system",
    content: [
      { type: "tool", name: "Read", description: "...", parameters: {...} },
      { type: "tool", name: "Edit", description: "...", parameters: {...} },
      ...
    ]
  },
  // 历史对话
  { role: "user", content: "请帮我修复这个 bug" },
  { role: "assistant", content: "我来帮你分析这个问题..." },
  { role: "user", content: "是的，继续" },
  { role: "assistant", content: [
      { type: "text", text: "我需要先读取文件..." },
      { type: "tool-call", id: "call_123", name: "Read", input: { path: "src/index.ts" } }
    ] 
  },
  // Tool 结果 (之前调用返回)
  { role: "tool", content: "文件内容: ...", toolCallId: "call_123" }
]
```

### 2.4 tools (工具定义)

```typescript
// AI SDK Tool 类型
type Tool = {
  description: string
  parameters: JSONSchema7
  execute: (input: any, options: ToolExecutionOptions) => Promise<ToolResult>
}

interface ToolExecutionOptions {
  toolCallId: string
  messages: Message[]
  abortSignal: AbortSignal
}

interface ToolResult {
  output: string
  title: string
  metadata?: Record<string, any>
  attachments?: FilePart[]
}
```

**内置工具示例**:
```typescript
{
  Read: {
    description: "Read the full contents of a file...",
    inputSchema: {
      type: "object",
      properties: {
        filePath: { type: "string", description: "..." }
      },
      required: ["filePath"]
    },
    execute: async (input, options) => {
      const content = await FileSystem.readText(input.filePath)
      return { output: content, title: "File Content", metadata: {} }
    }
  },
  Edit: {
    description: "Edit a file...",
    inputSchema: { type: "object", properties: {...}, required: [...] },
    execute: async (input, options) => { ... }
  },
  Write: { ... },
  Glob: { ... },
  Grep: { ... },
  Shell: { ... },
  // ... 更多工具
}
```

### 2.5 system (System Prompt)

```typescript
// 系统提示词数组，通常包含:
[
  "你是一个专业的 AI 编程助手...",  // Agent prompt
  "环境信息: ...",                   // sys.environment
  "可用工具: ...",                   // sys.tools (MCP tools)
  "项目信息: ...",                   // sys.project
  "指令: ..."                        // instruction.system
]
```

---

## 3. LLM API 返回 - StreamEvent 结构

### 3.1 事件类型

```typescript
// llm.ts:55
type Event = 
  | { type: "start" }
  | { type: "reasoning-start"; id: string; providerMetadata?: any }
  | { type: "reasoning-delta"; id: string; text: string; providerMetadata?: any }
  | { type: "reasoning-end"; id: string; providerMetadata?: any }
  | { type: "text-start"; providerMetadata?: any }
  | { type: "text-delta"; text: string; providerMetadata?: any }
  | { type: "text-end"; providerMetadata?: any }
  | { type: "tool-input-start"; id: string; toolName: string; providerMetadata?: any }
  | { type: "tool-input-delta"; id: string; text: string }
  | { type: "tool-input-end"; id: string; text: string }
  | { type: "tool-call"; toolCallId: string; toolName: string; input: any; providerMetadata?: any }
  | { type: "tool-result"; toolCallId: string; output: ToolResult; providerMetadata?: any }
  | { type: "tool-error"; toolCallId: string; error: Error }
  | { type: "start-step"; providerMetadata?: any }
  | { type: "finish-step"; finishReason: string; usage: Usage; providerMetadata?: any }
  | { type: "error"; error: Error }
  | { type: "finish" }
```

### 3.2 事件处理流程

```
┌────────────────────────────────────────────────────────────────────────────────────┐
│                              LLM API 事件流                                        │
└────────────────────────────────────────────────────────────────────────────────────┘
                                     │
      ┌──────────────────────────────┼──────────────────────────────┐
      │                              │                              │
      ▼                              ▼                              ▼
┌─────────────────┐        ┌─────────────────┐        ┌─────────────────┐
│  start-step     │        │  reasoning-*    │        │   text-*        │
│  步骤开始事件    │        │  推理过程事件    │        │  文本输出事件   │
└─────────────────┘        └─────────────────┘        └─────────────────┘
      │                              │                              │
      ▼                              ▼                              ▼
┌─────────────────┐        ┌─────────────────┐        ┌─────────────────┐
│ tool-input-start│        │ reasoning-start│        │ text-start      │
│ tool-call       │        │ reasoning-delta│        │ text-delta      │
│ tool-result     │        │ reasoning-end  │        │ text-end        │
│ tool-error      │        │                 │        │                 │
└─────────────────┘        └─────────────────┘        └─────────────────┘
      │                              │                              │
      └──────────────────────────────┴──────────────────────────────┘
                                     │
                                     ▼
┌────────────────────────────────────────────────────────────────────────────────────┐
│                          SessionProcessor.handleEvent()                            │
│  - 更新 MessageV2 消息部件 (part)                                                  │
│  - 触发 SessionEvent 事件                                                         │
│  - 处理工具执行结果                                                                │
└────────────────────────────────────────────────────────────────────────────────────┘
                                     │
                                     ▼
┌────────────────────────────────────────────────────────────────────────────────────┐
│                            finish-step 事件                                        │
│  - 更新消息完成状态                                                                │
│  - 计算 token 使用量                                                               │
│  - 检查是否需要压缩                                                                │
└────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 4. 完整 Session 交互示例

### 4.1 第一轮调用 (用户输入 → LLM)

```
┌────────────────────────────────────────────────────────────────────────────────────┐
│                              第一次 LLM 调用                                        │
├────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                    │
│ StreamInput:                                                                       │
│ {                                                                                 │
│   user: {                                                                         │
│     id: "msg_xxx",                                                                │
│     role: "user",                                                                 │
│     agent: "default",                                                            │
│     model: { providerID: "openai", modelID: "gpt-4o" },                          │
│     parts: [{ type: "text", text: "请帮我写一个 hello world 程序" }]             │
│   },                                                                              │
│   sessionID: "sess_xxx",                                                         │
│   model: { providerID: "openai", id: "gpt-4o", limit: {...} },                 │
│   agent: { name: "default", prompt: "...", permission: [...] },               │
│   system: ["你是一个 AI 编程助手", "可用工具: ..."],                              │
│   messages: [                                                                    │
│     { role: "system", content: "[工具定义]" },                                   │
│     { role: "user", content: "请帮我写一个 hello world 程序" }                  │
│   ],                                                                              │
│   tools: { Read: {...}, Write: {...}, Shell: {...} }                           │
│ }                                                                                 │
│                                                                                    │
└────────────────────────────────────────────────────────────────────────────────────┘
                                     │
                                     ▼
┌────────────────────────────────────────────────────────────────────────────────────┐
│                              LLM API 返回                                          │
├────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                    │
│ Event 序列:                                                                        │
│ 1. start-step                                                                      │
│ 2. text-start                                                                      │
│ 3. text-delta: "我来帮你创建一个 hello world 程序..."                           │
│ 4. text-end                                                                        │
│ 5. tool-input-start: { id: "call_1", toolName: "Write" }                         │
│ 6. tool-call: { toolCallId: "call_1", toolName: "Write",                        │
│                input: { filePath: "hello.py", content: "print('Hello World')" }}
│ 7. tool-result: { toolCallId: "call_1", output: { output: "成功", title: "..."}}
│ 8. finish-step: { finishReason: "tool-calls", usage: { input: 100, output: 50 }}
│                                                                                    │
└────────────────────────────────────────────────────────────────────────────────────┘
```

### 4.2 第二轮调用 (Tool Result → LLM)

```
┌────────────────────────────────────────────────────────────────────────────────────┐
│                              第二次 LLM 调用                                        │
├────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                    │
│ StreamInput:                                                                       │
│ {                                                                                 │
│   user: { ... },                                                                   │
│   messages: [                                                                    │
│     { role: "system", content: "[工具定义]" },                                   │
│     { role: "user", content: "请帮我写一个 hello world 程序" },                  │
│     { role: "assistant", content: "我来帮你..." },                               │
│     { role: "tool", content: "成功", toolCallId: "call_1" }  ← Tool Result      │
│   ],                                                                              │
│   tools: { ... },                                                                 │
│   ...                                                                             │
│ }                                                                                 │
│                                                                                    │
└────────────────────────────────────────────────────────────────────────────────────┘
                                     │
                                     ▼
┌────────────────────────────────────────────────────────────────────────────────────┐
│                              LLM API 返回                                          │
├────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                    │
│ Event 序列:                                                                        │
│ 1. start-step                                                                      │
│ 2. text-start                                                                      │
│ 3. text-delta: "我已经创建了 hello.py 文件，内容如下:\n\n```python\nprint('Hello World')\n```\n\n运行一下:" │
│ 4. tool-input-start: { id: "call_2", toolName: "Shell" }                         │
│ 5. tool-call: { toolCallId: "call_2", toolName: "Shell",                         │
│                input: { command: "python hello.py" }}                            │
│ 6. tool-result: { toolCallId: "call_2", output: { output: "Hello World", ...} }
│ 7. finish-step: { finishReason: "tool-calls", usage: {...} }                   │
│                                                                                    │
└────────────────────────────────────────────────────────────────────────────────────┘
```

### 4.3 第三轮调用 (继续对话)

```
┌────────────────────────────────────────────────────────────────────────────────────┐
│                              第三次 LLM 调用                                        │
├────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                    │
│ messages: [                                                                        │
│   { role: "system", content: "[工具定义]" },                                      │
│   { role: "user", content: "请帮我写一个 hello world 程序" },                    │
│   { role: "assistant", content: "我来帮你..." },                                 │
│   { role: "tool", content: "成功", toolCallId: "call_1" },                      │
│   { role: "assistant", content: "已经创建..." },                                │
│   { role: "tool", content: "Hello World", toolCallId: "call_2" }                │
│ ]                                                                                  │
│                                                                                    │
└────────────────────────────────────────────────────────────────────────────────────┘
                                     │
                                     ▼
┌────────────────────────────────────────────────────────────────────────────────────┐
│                              LLM API 返回                                          │
├────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                    │
│ Event 序列:                                                                        │
│ 1. start-step                                                                      │
│ 2. text-start                                                                      │
│ 3. text-delta: "程序已成功运行，输出了 'Hello World'！还有什么需要帮助的吗？"    │
│ 4. finish-step: { finishReason: "stop", usage: {...} }  ← 结束                  │
│ 5. finish                                                                         │
│                                                                                    │
└────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 5. 关键数据结构详解

### 5.1 MessageV2 转换为 ModelMessage

```typescript
// message-v2.ts
function toModelMessagesEffect(msgs: WithParts[]): Effect.Effect<ModelMessage[]> {
  return Effect.gen(function* () {
    const result: ModelMessage[] = []
    
    for (const msg of msgs) {
      if (msg.info.role === "user") {
        // 用户消息 → text + file parts
        const content = msg.parts.flatMap(part => {
          if (part.type === "text") 
            return [{ type: "text" as const, text: part.text }]
          if (part.type === "file") 
            return [{ type: "image" as const, image: part.url }]
          return []
        })
        result.push({ role: "user", content })
      }
      else if (msg.info.role === "assistant") {
        // 助手消息 → text + tool calls
        const content = msg.parts.flatMap(part => {
          if (part.type === "text") 
            return [{ type: "text" as const, text: part.text }]
          if (part.type === "tool" && part.state.status === "completed") {
            // 工具调用被转换为 tool-result
            return [{
              type: "tool-result" as const,
              toolCallId: part.callID,
              result: part.state.output
            }]
          }
          if (part.type === "tool" && part.state.status === "running") {
            // 正在运行的工具调用
            return [{
              type: "tool-call" as const,
              toolCallId: part.callID,
              toolName: part.tool,
              input: part.state.input
            }]
          }
          return []
        })
        if (content.length > 0) {
          result.push({ role: "assistant", content })
        }
      }
    }
    
    return result
  })
}
```

### 5.2 Token 计算

```typescript
// session.ts
interface Usage {
  input: number
  output: number
  reasoning: number
  cache: {
    read: number
    write: number
  }
  total: number
  cost: number
}

// 计算公式
function getUsage(model, usage, metadata) {
  const input = usage.promptTokens
  const output = usage.completionTokens
  const reasoning = usage.reasoningTokens ?? 0
  const cacheRead = usage.cacheReadTokens ?? 0
  const cacheWrite = usage.cacheWriteTokens ?? 0
  
  return {
    input,
    output,
    reasoning,
    cache: { read: cacheRead, write: cacheWrite },
    total: input + output + reasoning,
    cost: calculateCost(model, usage)
  }
}
```

---

## 6. 关键文件索引

| 文件 | 说明 |
|------|------|
| `packages/opencode/src/session/llm.ts` | LLM 服务，streamText 调用 |
| `packages/opencode/src/session/prompt.ts` | StreamInput 构建 |
| `packages/opencode/src/session/processor.ts` | 事件处理 |
| `packages/opencode/src/session/message-v2.ts` | MessageV2 数据模型 |
| `packages/opencode/src/session/system.ts` | System Prompt 构建 |
| `packages/opencode/src/tool/registry.ts` | 工具注册 |

---

## 7. 总结

### 7.1 数据流

1. **输入**: 用户消息 → 构建 StreamInput
2. **消息历史**: MessageV2 → ModelMessage[]
3. **工具定义**: ToolRegistry → AI SDK Tool[]
4. **调用**: streamText() → 事件流
5. **处理**: handleEvent() → 更新消息
6. **输出**: Tool Result → 下一轮调用或结束

### 7.2 关键点

- **消息格式转换**: MessageV2 → AI SDK ModelMessage
- **工具执行**: AI SDK 自动调用 tool.execute()
- **结果返回**: Tool result 作为消息发送回 LLM
- **循环控制**: finishReason 决定继续或结束