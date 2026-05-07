# Phase 9: 认证和权限系统 - 研究报告

> 研究时间: ~3 小时
> 状态: ✅ 已完成

## 1. 认证系统 (Auth)

### 1.1 核心功能

认证系统管理 OpenCode 与各 LLM Provider 之间的认证凭据。

**文件**: `src/auth/index.ts` (98 行)

### 1.2 支持的认证类型

```typescript
// OAuth 认证 (如 Google, Microsoft)
export class Oauth extends Schema.Class<Oauth>("OAuth")({
  type: Schema.Literal("oauth"),
  refresh: Schema.String,
  access: Schema.String,
  expires: NonNegativeInt,
  accountId: Schema.optional(Schema.String),
  enterpriseUrl: Schema.optional(Schema.String),
}) {}

// API Key 认证
export class Api extends Schema.Class<Api>("ApiAuth")({
  type: Schema.Literal("api"),
  key: Schema.String,
  metadata: Schema.optional(Schema.Record(Schema.String, Schema.String)),
}) {}

// Well-Known 认证
export class WellKnown extends Schema.Class<WellKnown>("WellKnownAuth")({
  type: Schema.Literal("wellknown"),
  key: Schema.String,
  token: Schema.String,
}) {}
```

### 1.3 认证接口

```typescript
export interface Interface {
  readonly get: (providerID: string) => Effect.Effect<Info | undefined, AuthError>
  readonly all: () => Effect.Effect<Record<string, Info>, AuthError>
  readonly set: (key: string, info: Info) => Effect.Effect<void, AuthError>
  readonly remove: (key: string) => Effect.Effect<void, AuthError>
}
```

### 1.4 数据存储

认证数据存储在 `~/.opencode/data/auth.json` (Linux/Mac) 或对应平台的数据目录。

```typescript
const file = path.join(Global.Path.data, "auth.json")
```

读取优先级:
1. 环境变量 `OPENCODE_AUTH_CONTENT` (JSON 字符串)
2. 文件 `auth.json`

### 1.5 权限模式

文件权限设置为 `0o600` (仅所有者可读写)，确保 API Key 安全。

---

## 2. 权限系统 (Permission)

### 2.1 核心概念

权限系统控制 Agent 可使用的工具和能力。

**文件**: `src/permission/index.ts` (324 行)

### 2.2 权限动作

```typescript
export const Action = Schema.Literals(["allow", "deny", "ask"])
export type Action = "allow" | "deny" | "ask"
```

| 动作 | 说明 |
|------|------|
| **allow** | 允许执行，无提示 |
| **deny** | 拒绝执行，抛异常 |
| **ask** | 询问用户确认 |

### 2.3 权限规则

```typescript
export const Rule = Schema.Struct({
  permission: Schema.String,  // 工具名称 (如 "read", "edit")
  pattern: Schema.String,     // 匹配模式 (如 "*.txt", "/home/*")
  action: Action,              // 动作
})
```

### 2.4 权限请求流程

```
工具执行请求
    │
    ▼
权限评估 (evaluate)
    │
    ├── deny ──► DeniedError
    │
    ├── allow ──► 继续执行
    │
    └── ask ──► 发布事件 → 等待用户响应
                    │
                    ├── once ──► 本次允许
                    ├── always ──► 永久允许 + 批量处理同类请求
                    └── reject ──► RejectedError
```

### 2.5 权限评估算法

**文件**: `src/permission/evaluate.ts` (15 行)

```typescript
export function evaluate(permission: string, pattern: string, ...rulesets: Rule[][]): Rule {
  const rules = rulesets.flat()
  const match = rules.findLast(
    (rule) => Wildcard.match(permission, rule.permission) && Wildcard.match(pattern, rule.pattern),
  )
  return match ?? { action: "ask", permission, pattern: "*" }
}
```

关键点:
- 使用 `findLast` 从后往前匹配，确保最新规则优先
- 支持通配符匹配 (`*`, `?`)
- 未匹配时默认 `ask`

### 2.6 权限数据结构

数据库存储在 `permission` 表:

```typescript
// src/session/session.sql.ts (行 125-131)
export const PermissionTable = sqliteTable("permission", {
  project_id: text().primaryKey().references(() => ProjectTable.id, { onDelete: "cascade" }),
  ...Timestamps,
  data: text({ mode: "json" }).notNull().$type<Permission.Ruleset>(),
})
```

### 2.7 权限请求消息

```typescript
export class Request extends Schema.Class<Request>("PermissionRequest")({
  id: PermissionID,
  sessionID: SessionID,
  permission: Schema.String,        // 权限类型
  patterns: Schema.Array(Schema.String), // 匹配模式
  metadata: Schema.Record(Schema.String, Schema.Unknown), // 额外信息
  always: Schema.Array(Schema.String),  // 自动授权的匹配项
  tool: Schema.optional(Schema.Struct({
    messageID: MessageID,
    callID: Schema.String,
  })),
})
```

### 2.8 工具执行时的权限检查

工具通过 `Tool.Context` 的 `ask` 方法请求权限:

```typescript
// src/tool/tool.ts (行 24)
export type Context<M extends Metadata = Metadata> = {
  // ...
  ask(input: Omit<Permission.Request, "id" | "sessionID" | "tool">): Effect.Effect<void>
}
```

示例 - 读取文件时请求权限:

```typescript
// src/tool/read.ts (行 179)
yield* ctx.ask({
  permission: "read",
  metadata: {},
  patterns: [target],
  always: [],
})
```

### 2.9 配置中的权限

在 `opencode.json` 或 Agent 配置中定义:

```json
{
  "permission": {
    "read": "allow",
    "edit": "ask",
    "bash": "deny",
    "glob": {
      "*": "allow",
      "~/.ssh/*": "deny"
    }
  }
}
```

支持的权限类型:
- `read` - 读取文件
- `edit` - 编辑/写入文件
- `glob` - 文件搜索
- `grep` - 内容搜索
- `list` - 列出目录
- `bash` - 执行命令
- `task` - 任务工具
- `webfetch` - Web 请求
- `websearch` - Web 搜索
- `skill` - 技能使用
- `lsp` - 语言服务器
- `todowrite` - 写 TODO
- `question` - 提问
- `doom_loop` - 死循环检测

### 2.10 权限事件

通过事件总线发布权限状态:

```typescript
export const Event = {
  Asked: BusEvent.define("permission.asked", Request),
  Replied: BusEvent.define(
    "permission.replied",
    Schema.Struct({
      sessionID: SessionID,
      requestID: PermissionID,
      reply: Reply,
    }),
  ),
}
```

---

## 3. 环境变量系统 (Env)

### 3.1 核心功能

管理每个项目实例的环境变量。

**文件**: `src/env/index.ts` (37 行)

### 3.2 环境变量接口

```typescript
export interface Interface {
  readonly get: (key: string) => Effect.Effect<string | undefined>
  readonly all: () => Effect.Effect<State>
  readonly set: (key: string, value: string) => Effect.Effect<void>
  readonly remove: (key: string) => Effect.Effect<void>
}
```

### 3.3 实现机制

使用 `InstanceState` 实现每个项目实例独立的环境变量:

```typescript
export const layer = Layer.effect(
  Service,
  Effect.gen(function* () {
    const state = yield* InstanceState.make<State>(
      Effect.fn("Env.state")(() => Effect.succeed({ ...process.env })),
    )
    // ...
  }),
)
```

关键点:
- 初始化时复制 `process.env`
- 每个项目实例有独立的环境变量副本
- 支持运行时修改环境变量

### 3.4 使用场景

**Provider 服务**: 使用 Env 获取模型供应商的环境变量

```typescript
// src/provider/provider.ts (行 1109-1110)
const dep = {
  auth: (id: string) => auth.get(id).pipe(Effect.orDie),
  config: () => config.get(),
  env: () => env.all(),
  get: (key: string) => env.get(key),
}
```

**配置服务**: 使用 Env 进行配置解析

```typescript
// src/config/config.ts (行 355)
const env = yield* Env.Service
```

---

## 4. 关键文件索引

| 文件 | 行数 | 说明 |
|------|------|------|
| `src/auth/index.ts` | 98 | 认证服务主文件 |
| `src/permission/index.ts` | 324 | 权限服务主文件 |
| `src/permission/evaluate.ts` | 15 | 权限评估逻辑 |
| `src/permission/schema.ts` | 16 | 权限相关 Schema |
| `src/permission/arity.ts` | - | 权限参数数量检查 |
| `src/config/permission.ts` | 70 | 权限配置解析 |
| `src/config/agent.ts` | 175 | Agent 配置 (含权限) |
| `src/env/index.ts` | 37 | 环境变量服务 |
| `src/session/session.sql.ts` | 131 | 数据库 Schema |

---

## 5. 系统交互图

```
┌─────────────────────────────────────────────────────────────┐
│                        配置层                                │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │ opencode.json│  │ Agent 配置   │  │ 环境变量            │  │
│  └──────┬──────┘  └──────┬──────┘  └──────────┬──────────┘  │
└─────────┼───────────────┼────────────────────┼───────────────┘
          │               │                    │
          ▼               ▼                    ▼
┌─────────────────────────────────────────────────────────────┐
│                     Config.Service                           │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ permission: fromConfig() → Ruleset                   │   │
│  │ env: Env.Service                                      │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                              │
          ┌───────────────────┼───────────────────┐
          ▼                   ▼                   ▼
┌─────────────────┐  ┌───────────────┐  ┌────────────────┐
│   Auth.Service  │  │Permission.Svc │  │  Env.Service   │
│                 │  │               │  │                │
│ - get/set/remove│  │ - ask()       │  │ - get/set/remove│
│ - OAuth/API Key │  │ - evaluate() │  │ - per-instance │
│ - auth.json     │  │ - reply()    │  │ - process.env  │
└─────────────────┘  └───────┬───────┘  └────────────────┘
                             │
                             ▼
                  ┌─────────────────────┐
                  │     Tool 执行       │
                  │                     │
                  │ ctx.ask() ──► 权限检查│
                  └─────────────────────┘
```

---

## 6. 待深入研究的问题

1. **OAuth 流程**: 具体如何处理 OAuth 回调和刷新令牌?
2. **权限持久化**: 权限规则如何跨会话持久化?
3. **权限优先级**: 当多个 Agent 配置冲突时如何处理?
4. **远程 Provider 认证**: 如何支持云端 Provider 的动态凭据?

---

## 7. 总结

Phase 9 完成了对认证和权限系统的深入理解:

- ✅ 理解了三种认证类型 (OAuth, API Key, Well-Known)
- ✅ 理解了权限评估机制 (allow/deny/ask)
- ✅ 理解了权限请求流程和用户交互
- ✅ 理解了环境变量的实例隔离机制

**关键发现**:
- Auth 系统使用文件存储，权限 0o600 保护
- Permission 使用通配符匹配，支持动态规则
- Env 使用 InstanceState 实现项目隔离
- 权限请求通过事件总线支持异步用户交互
- 工具通过 Tool.Context.ask() 请求权限

**下一步**: Phase 10 - 项目和会话管理