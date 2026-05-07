# Phase 11: 基础设施服务 - 研究报告

> 研究时间: ~2 小时
> 状态: ✅ 已完成

## 1. 事件总线 (Bus)

### 1.1 核心概念

事件总线是 OpenCode 的核心通信机制，支持实例内和全局事件发布/订阅。

**文件**: `src/bus/index.ts` (203 行)

### 1.2 事件定义

```typescript
// src/bus/bus-event.ts
export type Definition<Type extends string = string, Properties extends Schema.Top = Schema.Top> = {
  type: Type
  properties: Properties
}

export function define<Type extends string, Properties extends Schema.Top>(
  type: Type,
  properties: Properties,
): Definition<Type, Properties>
```

### 1.3 Bus 服务接口

```typescript
export interface Interface {
  readonly publish: <D extends BusEvent.Definition>(
    def: D,
    properties: BusProperties<D>,
    options?: { id?: string },
  ) => Effect.Effect<void>
  readonly subscribe: <D extends BusEvent.Definition>(def: D) => Stream.Stream<Payload<D>>
  readonly subscribeAll: () => Stream.Stream<Payload>
  readonly subscribeCallback: <D extends BusEvent.Definition>(
    def: D,
    callback: (event: Payload<D>) => unknown,
  ) => Effect.Effect<() => void>
  readonly subscribeAllCallback: (callback: (event: any) => unknown) => Effect.Effect<() => void>
}
```

### 1.4 实现机制

使用 Effect 的 `PubSub` 实现:

```typescript
type State = {
  wildcard: PubSub.PubSub<Payload>        // 订阅所有事件
  typed: Map<string, PubSub.PubSub<Payload>> // 按类型订阅
}
```

### 1.5 实例级事件

每个实例 (Project) 有独立的 Bus 状态:

```typescript
const state = yield* InstanceState.make<State>(
  Effect.fn("Bus.state")(function* (ctx) {
    const wildcard = yield* PubSub.unbounded<Payload>()
    const typed = new Map<string, PubSub.PubSub<Payload>>()
    // 清理时发布 InstanceDisposed 事件
    yield* Effect.addFinalizer(() =>
      Effect.gen(function* () {
        yield* PubSub.publish(wildcard, {
          type: InstanceDisposed.type,
          id: createID(),
          properties: { directory: ctx.directory },
        })
        yield* PubSub.shutdown(wildcard)
      }),
    )
    return { wildcard, typed }
  }),
)
```

### 1.6 全局事件 (GlobalBus)

跨实例的事件广播:

```typescript
// src/bus/global.ts
class GlobalBusEmitter extends EventEmitter<{
  event: [GlobalEvent]
}> {}

export const GlobalBus = new GlobalBusEmitter()
```

### 1.7 事件示例

```typescript
// 权限事件
export const Event = {
  Asked: BusEvent.define("permission.asked", Request),
  Replied: BusEvent.define("permission.replied", Schema.Struct({
    sessionID: SessionID,
    requestID: PermissionID,
    reply: Reply,
  })),
}

// 项目事件
export const Event = {
  Updated: BusEvent.define("project.updated", Info),
}

// LSP 事件
export const Event = {
  Updated: BusEvent.define("lsp.updated", Schema.Struct({})),
}
```

---

## 2. 日志系统

### 2.1 日志创建

**来源**: `@opencode-ai/core/util/log`

```typescript
const log = Log.create({ service: "permission" })

// 使用方式
log.info("publishing", { type: def.type })
log.error("subscriber failed", { type, cause })
log.clone().tag("session.id", input.sessionID).tag("messageID", input.assistantMessage.id)
```

### 2.2 日志级别

- `info` - 信息日志
- `warning` - 警告日志
- `error` - 错误日志

### 2.3 日志标签

支持添加标签用于追踪:

```typescript
const slog = log.clone().tag("session.id", input.sessionID).tag("messageID", input.assistantMessage.id)
```

---

## 3. 错误处理

### 3.1 错误格式化

**文件**: `src/util/error.ts` (88 行)

```typescript
export function errorFormat(error: unknown): string {
  if (error instanceof Error) {
    return error.stack ?? `${error.name}: ${error.message}`
  }
  // 处理对象错误
  if (typeof error === "object" && error !== null) {
    try {
      return JSON.stringify(error, null, 2)
    } catch {
      return "Unexpected error (unserializable)"
    }
  }
  return String(error)
}
```

### 3.2 错误消息提取

```typescript
export function errorMessage(error: unknown): string {
  if (error instanceof Error) {
    if (error.message) return error.message
    if (error.name) return error.name
  }
  // 处理带 message 属性的对象
  if (isRecord(error) && typeof error.message === "string" && error.message) {
    return error.message
  }
  // 处理嵌套 data.message
  if (isRecord(error) && isRecord(error.data) && typeof error.data.message === "string") {
    return error.data.message
  }
  return "unknown error"
}
```

### 3.3 错误数据提取

```typescript
export function errorData(error: unknown) {
  if (error instanceof Error) {
    return {
      type: error.name,
      message: errorMessage(error),
      stack: error.stack,
      cause: error.cause === undefined ? undefined : errorFormat(error.cause),
      formatted: errorFormat(error),
    }
  }
  // 处理普通对象
  // ...
}
```

### 3.4 命名错误 (NamedError)

```typescript
// 来自 @opencode-ai/core/util/error
export const NotGitError = NamedError.create(
  "WorktreeNotGitError",
  z.object({
    message: z.string(),
  }),
)

export const CreateFailedError = NamedError.create(
  "WorktreeCreateFailedError",
  z.object({
    message: z.string(),
  }),
)
```

### 3.5 Schema 错误

使用 Effect 的 `Schema.TaggedErrorClass`:

```typescript
export class AuthError extends Schema.TaggedErrorClass<AuthError>()("AuthError", {
  message: Schema.String,
  cause: Schema.optional(Schema.Defect),
}) {}

export class DeniedError extends Schema.TaggedErrorClass<DeniedError>()("PermissionDeniedError", {
  ruleset: Schema.Any,
}) {
  override get message() {
    return `The user has specified a rule which prevents you from using this specific tool call.`
  }
}
```

---

## 4. 关键文件索引

| 文件 | 行数 | 说明 |
|------|------|------|
| `src/bus/index.ts` | 203 | Bus 服务主文件 |
| `src/bus/bus-event.ts` | 51 | 事件定义工具 |
| `src/bus/global.ts` | 22 | 全局事件总线 |
| `src/util/error.ts` | 88 | 错误处理工具 |
| `@opencode-ai/core/util/log.ts` | - | 日志系统 |

---

## 5. 系统交互图

```
┌─────────────────────────────────────────────────────────────┐
│                      事件发布                               │
│                                                              │
│  Bus.publish(Event, Properties)                            │
│         │                                                   │
│         ├──► typed PubSub ──► 订阅者                       │
│         │                                                   │
│         ├──►  wildcard PubSub ──► 全订阅者                  │
│         │                                                   │
│         └──► GlobalBus.emit() ──► 跨实例事件                │
└─────────────────────────────────────────────────────────────┘
                              │
          ┌───────────────────┼───────────────────┐
          ▼                   ▼                   ▼
┌─────────────────┐  ┌───────────────┐  ┌────────────────┐
│   权限系统      │  │   项目系统    │  │   会话系统     │
│                 │  │               │  │                │
│ permission.ask │  │ project.updated│ │ session.error  │
│ permission.repl│  │                │  │                │
└─────────────────┘  └───────────────┘  └────────────────┘
```

---

## 6. 总结

Phase 11 完成了对基础设施服务的深入理解:

- ✅ 理解了事件总线机制 (PubSub + EventEmitter)
- ✅ 理解了实例级和全局事件
- ✅ 理解了日志系统
- ✅ 理解了错误处理和格式化

**关键发现**:
- 使用 Effect PubSub 实现类型安全的事件订阅
- GlobalBus 支持跨实例通信
- 错误处理支持多种格式 (Error, 对象, 字符串)
- 日志支持标签追踪

**下一步**: Phase 12 - Git 和 IDE 集成