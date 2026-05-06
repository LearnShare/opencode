# ACP (Agent Client Protocol) 协议研究报告

---

## 1. 概述

ACP (Agent Client Protocol) 是 OpenCode 作为 Agent 服务端的协议实现，允许外部客户端通过标准协议与 OpenCode 交互。

### 1.1 基本架构

```
外部客户端 ←→ ACP Server (opencode acp) ←→ OpenCode Session ←→ LLM
```

### 1.2 依赖包

- `@agentclientprotocol/sdk` - ACP SDK
- `@opencode-ai/sdk/v2` - OpenCode SDK

---

## 2. 服务启动

### 2.1 命令

```bash
opencode acp [--cwd <directory>]
```

### 2.2 启动流程

1. 启动 HTTP Server
2. 创建 OpenCodeClient SDK 实例
3. 设置 stdin/stdout NDJSON 流
4. 初始化 Agent 并建立连接

```typescript
// src/cli/cmd/acp.ts
const server = yield* Effect.promise(() => Server.listen(opts))
const sdk = createOpencodeClient({ baseUrl: `http://${server.hostname}:${server.port}`, ... })

const stream = ndJsonStream(input, output)
const agent = ACP.init({ sdk })

new AgentSideConnection((conn) => {
  return agent.create(conn, { sdk })
}, stream)
```

---

## 3. ACP 客户端连接

### 3.1 连接建立

```typescript
// src/acp/agent.ts
export function init({ sdk }: { sdk: OpencodeClient }) {
  return {
    create: (connection: AgentSideConnection, fullConfig: ACPConfig) => {
      return new Agent(connection, fullConfig)
    },
  }
}

export class Agent implements ACPAgent {
  private connection: AgentSideConnection
  private config: ACPConfig
  private sdk: OpencodeClient
  private sessionManager: ACPSessionManager
  
  constructor(connection: AgentSideConnection, config: ACPConfig) {
    this.connection = connection
    this.config = config
    this.sdk = config.sdk
    this.sessionManager = new ACPSessionManager(this.sdk)
    this.startEventSubscription()
  }
}
```

---

## 4. 会话管理

### 4.1 会话状态

```typescript
interface ACPSessionState {
  id: string           // 会话 ID
  cwd: string          // 工作目录
  mcpServers: McpServer[]  // MCP 服务器配置
  createdAt: Date     // 创建时间
  model?: {           // 当前模型
    providerID: ProviderID
    modelID: ModelID
  }
  variant?: string     // 模型变体
  modeId?: string      // 会话模式
}
```

### 4.2 会话管理器

```typescript
class ACPSessionManager {
  private sessions = new Map<string, ACPSessionState>()
  private sdk: OpencodeClient
  
  // 创建新会话
  async create(cwd: string, mcpServers: McpServer[], model?): Promise<ACPSessionState>
  
  // 加载已有会话
  async load(sessionId: string, cwd: string, mcpServers: McpServer[], model?): Promise<ACPSessionState>
  
  // 获取会话
  get(sessionId: string): ACPSessionState
  
  // 模型/变体/模式管理
  getModel(sessionId: string): ACPSessionState["model"]
  setModel(sessionId: string, model: ACPSessionState["model"])
  getVariant(sessionId: string): string | undefined
  setVariant(sessionId: string, variant?: string)
  setMode(sessionId: string, modeId: string)
}
```

---

## 5. 协议请求处理

### 5.1 核心请求类型

| 请求 | 说明 |
|------|------|
| initialize | 初始化连接 |
| newSession | 创建新会话 |
| loadSession | 加载已有会话 |
| resumeSession | 恢复会话 |
| listSessions | 列出所有会话 |
| forkSession | Fork 会话 |
| setSessionModel | 设置会话模型 |
| setSessionMode | 设置会话模式 |
| setSessionConfigOption | 设置配置选项 |
| sendMessage | 发送消息 |
| cancel | 取消执行 |
| authenticate | 认证 |
| requestPermission | 请求权限 |

### 5.2 初始化

```typescript
async initialize(request: InitializeRequest): Promise<InitializeResponse> {
  return {
    protocolVersion: "1.0",
    capabilities: {
      tools: true,
      prompts: true,
      resources: true,
      sessions: true,
    },
    serverInfo: {
      name: "opencode",
      version: InstallationVersion,
    },
    options: {
      modes: modeOptions,
      models: modelOptions,
    }
  }
}
```

### 5.3 会话创建

```typescript
async newSession(params: NewSessionRequest): Promise<NewSessionResponse> {
  const directory = params.cwd
  const mcpServers = params.mcpServers ?? []
  const model = await defaultModel(this.config, directory)
  
  const state = await this.sessionManager.create(directory, mcpServers, model)
  const result = await this.loadSessionMode({ cwd: directory, mcpServers, sessionId: state.id })
  
  return result
}
```

### 5.4 消息处理

```typescript
async sendMessage(params: {
  sessionId: string
  message: { role: "user" | "assistant"; content: string }
}): Promise<void> {
  // 发送消息到 OpenCode 会话
  await this.sdk.session.sendMessage({
    sessionID: params.sessionId,
    message: params.message.content,
    directory: session.cwd,
  })
}
```

---

## 6. 事件订阅与推送

### 6.1 事件订阅

```typescript
private startEventSubscription() {
  this.runEventSubscription().catch((error) => {
    log.error("event subscription failed", { error })
  })
}

private async runEventSubscription() {
  while (true) {
    const events = await this.sdk.global.event({ signal: this.eventAbort.signal })
    for await (const event of events.stream) {
      await this.handleEvent(payload as Event).catch((error) => {
        log.error("failed to handle event", { error })
      })
    }
  }
}
```

### 6.2 处理的事件

| 事件 | 说明 |
|------|------|
| permission.asked | 权限请求 |
| message.part.updated | 消息部件更新 (tool/text) |

### 6.3 推送更新

```typescript
// 工具调用更新
await this.connection.sessionUpdate({
  sessionId,
  update: {
    sessionUpdate: "tool_call_update",
    toolCallId: part.callID,
    status: "in_progress" | "completed" | "failed",
    kind: toToolKind(part.tool),
    title: part.tool,
    content: [...],
    rawInput: part.state.input,
    rawOutput: part.state.output,
  },
})

// 文本消息块
await this.connection.sessionUpdate({
  sessionId,
  update: {
    sessionUpdate: "agent_message_chunk" | "user_message_chunk",
    messageId: message.info.id,
    content: { type: "text", text: part.text },
  },
})

// 使用量更新
await this.connection.sessionUpdate({
  sessionId,
  update: {
    sessionUpdate: "usage_update",
    used,
    size,
    cost: { amount: totalCost, currency: "USD" },
  },
})

// Plan 更新 (TodoWrite)
await this.connection.sessionUpdate({
  sessionId,
  update: {
    sessionUpdate: "plan",
    entries: [...],
  },
})
```

---

## 7. 权限处理

### 7.1 权限选项

```typescript
private permissionOptions: PermissionOption[] = [
  { optionId: "once", kind: "allow_once", name: "Allow once" },
  { optionId: "always", kind: "allow_always", name: "Always allow" },
  { optionId: "reject", kind: "reject_once", name: "Reject" },
]
```

### 7.2 处理流程

1. 接收 `permission.asked` 事件
2. 向 ACP 客户端请求权限
3. 用户选择后回复 OpenCode

---

## 8. 关键文件

| 文件 | 说明 |
|------|------|
| `src/cli/cmd/acp.ts` | ACP 命令入口 (73 行) |
| `src/acp/agent.ts` | ACP Agent 实现 (1842 行) |
| `src/acp/session.ts` | 会话管理器 (116 行) |
| `src/acp/types.ts` | 类型定义 (24 行) |

---

## 9. 总结

- **协议**: 基于 `@agentclientprotocol/sdk`
- **传输**: NDJSON over stdio
- **会话管理**: 创建/加载/恢复/Fork
- **事件驱动**: 订阅 OpenCode 事件，推送到 ACP 客户端
- **权限桥接**: ACP 权限请求 ↔ OpenCode 权限系统