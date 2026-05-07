# MCP (Model Context Protocol) 集成研究报告

> 研究时间: ~4 小时
> 状态: ✅ 已完成
> 关联 Phase: 5

---

## 1. MCP 配置格式

### 1.1 本地 MCP 服务器

```typescript
{
  "type": "local",
  "command": ["npx", "-y", "@modelcontextprotocol/server-filesystem", "/path/to/dir"],
  "environment": { "KEY": "value" },  // 可选，环境变量
  "enabled": true,                     // 可选，是否启用
  "timeout": 30000                      // 可选，请求超时 (ms)
}
```

### 1.2 远程 MCP 服务器

```typescript
{
  "type": "remote",
  "url": "https://mcp.example.com/server",
  "headers": { "Authorization": "Bearer token" },  // 可选，自定义请求头
  "oauth": {                    // 可选，OAuth 配置
    "clientId": "xxx",         // 可选，预注册的 client ID
    "clientSecret": "xxx",     // 可选，client secret
    "scope": "read write",      // 可选，OAuth  scopes
    "redirectUri": "http://127.0.0.1:19876/mcp/oauth/callback"  // 可选
  },
  "enabled": true,
  "timeout": 30000
}
```

### 1.3 配置位置

配置文件: `opencode.json`

```json
{
  "mcp": {
    "my-server": {
      "type": "remote",
      "url": "https://..."
    }
  }
}
```

---

## 2. 连接类型与传输方式

### 2.1 连接类型

| 类型 | 说明 | 使用场景 |
|------|------|----------|
| local | 通过 stdio 启动子进程 | 本地运行的 MCP 服务器 |
| remote | 通过 HTTP 连接 | 远程 MCP 服务器 |

### 2.2 传输协议 (远程)

| 协议 | 说明 | 优先级 |
|------|------|--------|
| StreamableHTTP | MCP SDK 原生 HTTP 流式传输 | 优先尝试 |
| SSE | Server-Sent Events | 备用 |

### 2.3 本地传输

- 使用 `StdioClientTransport`
- 通过子进程 stdio 通信
- 支持 stderr 日志输出

---

## 3. MCP 服务状态

### 3.1 状态类型

| 状态 | 说明 |
|------|------|
| connected | 已连接并正常工作 |
| disabled | 配置中禁用 |
| failed | 连接失败 |
| needs_auth | 需要 OAuth 认证 |
| needs_client_registration | 需要预注册的 client ID |

### 3.2 状态存储

状态存储在 `InstanceState` 中:

```typescript
interface State {
  status: Record<string, Status>      // 每个 MCP 服务器的状态
  clients: Record<string, MCPClient>   // 活跃的客户端实例
  defs: Record<string, MCPToolDef[]>  // 缓存的工具定义
}
```

---

## 4. 工具与资源

### 4.1 工具定义格式

MCP 工具转换为 AI SDK Tool 格式:

```typescript
{
  name: string,           // 工具名称
  description: string,  // 工具描述
  inputSchema: JSONSchema7  // 输入参数 schema
}
```

### 4.2 工具名称格式

```
{sanitized_client_name}_{sanitized_tool_name}
```

例如: `filesystem_read` (如果客户端名为 "filesystem")

### 4.3 工具调用结果

使用 `CallToolResultSchema` 标准化结果格式

### 4.4 资源类型

```typescript
{
  name: string,           // 资源名称
  uri: string,            // 资源 URI
  description?: string,   // 可选描述
  mimeType?: string,      // 可选 MIME 类型
  client: string          // 所属客户端名称
}
```

### 4.5 提示词类型

MCP 服务器提供的提示词模板

---

## 5. OAuth 认证

### 5.1 认证状态

| 状态 | 说明 |
|------|------|
| authenticated | 已认证且 token 有效 |
| expired | token 已过期 |
| not_authenticated | 未认证 |

### 5.2 认证数据存储

文件: `$DATA/mcp-auth.json`

```typescript
{
  "mcp-server-name": {
    tokens: {
      accessToken: string,
      refreshToken?: string,
      expiresAt?: number,
      scope?: string
    },
    clientInfo?: {
      clientId: string,
      clientSecret?: string,
      clientIdIssuedAt?: number,
      clientSecretExpiresAt?: number
    },
    codeVerifier?: string,
    oauthState?: string,
    serverUrl?: string
  }
}
```

### 5.3 OAuth 流程

1. **动态客户端注册**: 如果服务器支持，自动注册客户端
2. **预注册客户端**: 如果配置了 clientId 和 clientSecret，使用预注册方式
3. **Callback 机制**: 本地回调服务器在 `http://127.0.0.1:19876/mcp/oauth/callback`

---

## 6. 服务接口

### 6.1 MCP Service 接口

```typescript
interface Interface {
  // 状态查询
  status(): Effect.Effect<Record<string, Status>>
  clients(): Effect.Effect<Record<string, MCPClient>>
  
  // 工具与资源
  tools(): Effect.Effect<Record<string, Tool>>
  prompts(): Effect.Effect<Record<string, PromptInfo & { client: string }>>
  resources(): Effect.Effect<Record<string, ResourceInfo & { client: string }>>
  
  // 连接管理
  add(name: string, mcp: ConfigMCP.Info): Effect.Effect<{ status: Record<string, Status> | Status }>
  connect(name: string): Effect.Effect<void>
  disconnect(name: string): Effect.Effect<void>
  
  // Prompt 与资源访问
  getPrompt(clientName: string, name: string, args?: Record<string, string>): Effect.Effect<PromptResult | undefined>
  readResource(clientName: string, resourceUri: string): Effect.Effect<ResourceResult | undefined>
  
  // OAuth 认证
  startAuth(mcpName: string): Effect.Effect<{ authorizationUrl: string; oauthState: string }>
  authenticate(mcpName: string): Effect.Effect<Status>
  finishAuth(mcpName: string, authorizationCode: string): Effect.Effect<Status>
  removeAuth(mcpName: string): Effect.Effect<void>
  supportsOAuth(mcpName: string): Effect.Effect<boolean>
  hasStoredTokens(mcpName: string): Effect.Effect<boolean>
  getAuthStatus(mcpName: string): Effect.Effect<AuthStatus>
}
```

---

## 7. 事件通知

### 7.1 工具列表变更

```typescript
// 事件: MCP 服务器工具列表变化
BusEvent.define(
  "mcp.tools.changed",
  Schema.Struct({ server: Schema.String })
)
```

当 MCP 服务器工具列表变化时触发，通知 UI 更新

### 7.2 浏览器打开失败

```typescript
BusEvent.define(
  "mcp.browser.open.failed",
  Schema.Struct({ mcpName: Schema.String, url: Schema.String })
)
```

---

## 8. 关键文件索引

| 文件 | 说明 |
|------|------|
| `src/mcp/index.ts` | MCP 核心实现 (931 行) |
| `src/mcp/auth.ts` | OAuth 认证存储 (144 行) |
| `src/mcp/oauth-provider.ts` | OAuth 提供者实现 (214 行) |
| `src/mcp/oauth-callback.ts` | OAuth 回调处理 |
| `src/config/mcp.ts` | MCP 配置 Schema (65 行) |

---

## 9. 依赖包

| 包 | 说明 |
|---|------|
| `@modelcontextprotocol/sdk` | MCP SDK (client) |
| `ai` | AI SDK (工具定义) |

---

## 10. 总结

- **两种连接方式**: local (stdio) 和 remote (HTTP)
- **两种传输协议**: StreamableHTTP (优先) 和 SSE
- **工具转换**: MCP Tool → AI SDK Tool
- **OAuth 支持**: 动态注册 + 预注册客户端
- **状态管理**: 连接状态、工具定义缓存
- **事件驱动**: 工具列表变更通知