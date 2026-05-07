# 每轮 LLM 交互的完整数据内容

> 本文档详细说明在多轮对话中，每一轮向 LLM 发送的具体内容。

---

## 1. 发送给 LLM 的完整数据结构

在 `prompt.ts:1577-1588` 中，每次 LLM 调用的输入:

```typescript
const result = yield* handle.process({
  user: lastUser,           // 当前用户消息
  agent,                    // Agent 信息
  permission: session.permission,  // 权限规则
  sessionID,
  parentSessionID: session.parentID,
  system,                   // System Prompt 数组
  messages: [...modelMsgs, ...maxStepsMsg],  // 历史消息
  tools,                    // 工具定义
  model,                    // 模型信息
  toolChoice: ...,
})
```

---

## 2. 第一轮 - 用户首次输入

### 2.1 messages 数组内容

假设用户输入: "帮我写一个 hello world 程序"

```
messages = [
  // System 消息 (由 system 参数提供)
]
```

### 2.2 system 数组内容 (prompt.ts:1574)

```typescript
const system = [...env, ...instructions, ...(skills ? [skills] : [])]
```

- **env** (sys.environment): 操作系统、工作目录、可用工具等环境信息
- **instructions**: 指令 (如 AGENTS.md 中的自定义指令)
- **skills**: Skill 工具描述 (如果有)

### 2.3 tools 数组内容

由 `resolveTools()` 构建，包含所有可用工具:

```typescript
tools = {
  Read: {
    description: "Read the full contents of a file...",
    inputSchema: { type: "object", properties: { filePath: { type: "string" } }, required: ["filePath"] },
    execute: async (input, options) => { ... }
  },
  Write: {
    description: "Write content to a file at the given path...",
    inputSchema: { type: "object", properties: { filePath: { type: "string" }, content: { type: "string" } }, required: ["filePath", "content"] },
    execute: async (input, options) => { ... }
  },
  Edit: { ... },
  Glob: { ... },
  Grep: { ... },
  Shell: { ... },
  Task: { ... },
  // ... 更多内置工具
}
```

---

## 3. 第二轮 - LLM 返回工具调用，工具执行完成后

### 3.1 messages 数组内容

在第二轮调用前，消息链已更新为:

```
messages = [
  // 1. 用户原始消息 (role: "user")
  {
    role: "user",
    content: "帮我写一个 hello world 程序"
  },

  // 2. LLM 第一次响应 (role: "assistant")
  // 包含 text + tool-call
  {
    role: "assistant",
    content: [
      { type: "text", text: "我来帮你创建这个程序..." },
      {
        type: "tool-call",
        toolCallId: "call_abc123",
        toolName: "Write",
        input: { filePath: "hello.py", content: "print('Hello World')" }
      }
    ]
  },

  // 3. 工具执行结果 (role: "tool")
  // 这是关键: tool result 作为独立的 user 消息
  {
    role: "tool",
    content: "文件已创建: hello.py",
    toolCallId: "call_abc123"
  }
]
```

### 3.2 转换过程 (message-v2.ts:791-979)

```typescript
for (const msg of input) {
  // 用户消息
  if (msg.info.role === "user") {
    result.push({
      role: "user",
      parts: [
        { type: "text", text: "帮我写一个 hello world 程序" }
        // 如果有文件附件: { type: "file", url: "...", mediaType: "..." }
      ]
    })
  }

  // 助手消息 - 包含 text + tool-call
  if (msg.info.role === "assistant") {
    result.push({
      role: "assistant",
      parts: [
        { type: "text", text: "我来帮你创建这个程序..." },
        // tool-call 保留在 assistant 消息中
        {
          type: "tool-call",
          toolCallId: "call_abc123",
          toolName: "Write",
          input: { filePath: "hello.py", content: "..." }
        }
      ]
    })
  }

  // 工具结果 - 作为独立的 "tool" 角色消息
  // 注意: 这里遍历所有消息，将 tool result 追加为独立消息
  for (const part of msg.parts) {
    if (msg.info.role === "assistant" && part.type === "tool" && part.state.status === "completed") {
      result.push({
        role: "tool",
        content: part.state.output,  // "文件已创建: hello.py"
        toolCallId: part.callID
      })
    }
  }
}
```

---

## 4. 第三轮 - 继续对话

### 4.1 消息链

假设用户回复: "运行一下"

```
messages = [
  // 第一轮
  { role: "user", content: "帮我写一个 hello world 程序" },
  { role: "assistant", content: ["我来帮你...", tool-call: Write] },
  { role: "tool", content: "文件已创建", toolCallId: "call_1" },

  // 第二轮
  { role: "assistant", content: ["已创建，运行一下:", tool-call: Shell] },
  { role: "tool", content: "Hello World\n", toolCallId: "call_2" },

  // 第三轮 (用户新输入)
  { role: "user", content: "运行一下" },

  // 注意: 上一轮 assistant 的 text 部分会作为当前轮的上下文
]
```

---

## 5. 完整消息流示例

### 5.1 场景: 用户要求写程序并运行

**第一轮调用**:
```
LLM Input:
{
  system: ["环境信息...", "你是一个编程助手", "可用工具: Read, Write, Shell..."],
  messages: [
    { role: "user", content: "帮我写一个 hello world 程序" }
  ],
  tools: { Read, Write, Edit, Glob, Grep, Shell, ... }
}

LLM Output:
{
  text: "我来帮你创建 hello.py 文件...",
  tool_calls: [{ id: "call_1", name: "Write", input: { filePath: "hello.py", content: "..." } }]
}
```

**工具执行**:
- 执行 Write 工具
- 结果: "文件已创建"

**第二轮调用** (tool result 发送回 LLM):
```
LLM Input:
{
  system: ["环境信息...", "你是一个编程助手", ...],
  messages: [
    { role: "user", content: "帮我写一个 hello world 程序" },

    { role: "assistant", content: [
      { type: "text", text: "我来帮你创建 hello.py 文件..." },
      { type: "tool-call", id: "call_1", name: "Write", input: { filePath: "hello.py", ... } }
    ]},

    { role: "tool", content: "文件已创建: hello.py", toolCallId: "call_1" }
  ],
  tools: { Read, Write, ... }
}

LLM Output:
{
  text: "文件已创建，现在运行它...",
  tool_calls: [{ id: "call_2", name: "Shell", input: { command: "python hello.py" } }]
}
```

**第三轮调用**:
```
LLM Input:
{
  messages: [
    { role: "user", content: "帮我写一个 hello world 程序" },
    { role: "assistant", content: [...] + tool-call: Write },
    { role: "tool", content: "文件已创建", toolCallId: "call_1" },

    { role: "assistant", content: [
      { type: "text", text: "文件已创建，现在运行它..." },
      { type: "tool-call", id: "call_2", name: "Shell", input: { command: "python hello.py" } }
    ]},

    { role: "tool", content: "Hello World\n", toolCallId: "call_2" }
  ],
  ...
}

LLM Output:
{
  text: "程序成功运行，输出了 'Hello World'！还有其他需要帮助的吗？",
  finish_reason: "stop"
}
```

---

## 6. 关键转换点

### 6.1 MessageV2 → ModelMessage 转换规则

| MessageV2 | ModelMessage | 说明 |
|-----------|--------------|------|
| User msg + TextPart | `{ role: "user", content: [...] }` | 用户消息 |
| User msg + FilePart | `{ role: "user", content: [{ type: "file", ... }] }` | 带附件 |
| User msg + CompactionPart | `{ role: "user", content: "What did we do so far?" }` | 压缩摘要 |
| Assistant msg + TextPart | `{ role: "assistant", content: [...] }` | 助手文本 |
| Assistant msg + ToolPart (running) | `{ role: "assistant", content: [{ type: "tool-call", ... }] }` | 工具调用 |
| Assistant msg + ToolPart (completed) | `{ role: "tool", content: "...", toolCallId: "..." }` | **独立消息** |
| Assistant msg + ToolPart (error) | `{ role: "tool", content: error, toolCallId: "..." }` | 错误结果 |
| Assistant msg + ToolPart (pending) | `{ role: "assistant", content: [{ type: "tool-call", error: "interrupted" }] }` | 中断 |

### 6.2 工具结果为什么是独立消息

这是 AI SDK (Vercel) 的设计:

- **tool-call** 保留在 assistant 消息中，表示"我要调用工具"
- **tool result** 必须作为独立的 `role: "tool"` 消息，表示"工具返回了结果"

这样 LLM 可以清楚地看到:
1. 助手说了一些话
2. 助手调用了工具
3. 工具返回了结果
4. 基于结果，助手继续说

---

## 7. 多轮后的完整消息数组

假设经过 5 轮对话后:

```typescript
// 实际发送给 LLM 的 messages 数组
[
  // ====== 第一轮 ======
  { role: "user", content: "帮我写一个 hello world 程序" },

  { role: "assistant", content: [
    { type: "text", text: "我来帮你创建这个程序..." },
    { type: "tool-call", toolCallId: "c1", name: "Write", input: { filePath: "hello.py", content: "print('Hello World')" } }
  ]},
  { role: "tool", content: "文件已创建: hello.py", toolCallId: "c1" },

  // ====== 第二轮 ======
  { role: "assistant", content: [
    { type: "text", text: "文件已创建，现在运行它..." },
    { type: "tool-call", toolCallId: "c2", name: "Shell", input: { command: "python hello.py" } }
  ]},
  { role: "tool", content: "Hello World\n", toolCallId: "c2" },

  // ====== 第三轮 ======
  { role: "assistant", content: [
    { type: "text", text: "运行成功！输出 'Hello World'" }
  ]},
  // 这轮没有工具调用，finish_reason = "stop"

  // ====== 用户新输入 ======
  { role: "user", content: "再运行一次" },

  // ====== 第四轮 ======
  { role: "assistant", content: [
    { type: "text", text: "好的，再运行一次..." },
    { type: "tool-call", toolCallId: "c3", name: "Shell", input: { command: "python hello.py" } }
  ]},
  { role: "tool", content: "Hello World\n", toolCallId: "c3" },

  // ====== 第五轮 ======
  { role: "assistant", content: [
    { type: "text", text: "又成功运行了一次！还有其他需要吗？" }
  ]},
  // finish_reason = "stop"
]
```

---

## 8. system 参数详解

每一轮都重新构建 system:

```typescript
// prompt.ts:1568-1576
const [skills, env, instructions, modelMsgs] = yield* Effect.all([
  sys.skills(agent),           // Skill 工具描述
  sys.environment(model),       // 环境信息
  instruction.system().pipe(Effect.orDie),  // 指令 (AGENTS.md)
  MessageV2.toModelMessagesEffect(msgs, model),
])
const system = [...env, ...instructions, ...(skills ? [skills] : [])]
```

**注意**: system 数组中的内容在每轮可能变化 (如动态加载的 MCP 工具)，但历史消息 `messages` 数组会不断累积。

---

## 9. 总结

### 每一轮发送的核心内容

| 字段 | 来源 | 说明 |
|------|------|------|
| `system` | 每轮重新构建 | 环境 + 指令 + 技能 |
| `messages` | 累积所有历史 | user → assistant → tool → user → assistant → ... |
| `tools` | 每轮重新构建 | 所有可用工具定义 |
| `user` | 当前用户消息 | 当前输入 |
| `model` | 从 session 获取 | 使用的模型 |
| `agent` | 从 session 获取 | Agent 配置 |

### 消息累积规则

1. **user 消息**: 用户输入
2. **assistant 消息**: LLM 回复 (可能包含 text + tool-call)
3. **tool 消息**: 工具执行结果 (作为独立消息插在 assistant 后面)
4. **循环**: 重复 2-3 直到 finish_reason = "stop" 且无 tool-calls

这样确保 LLM 始终有完整上下文，可以基于历史对话和工具结果做出正确决策。