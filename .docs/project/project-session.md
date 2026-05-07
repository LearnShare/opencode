# Phase 10: 项目和会话管理 - 研究报告

> 研究时间: ~4 小时
> 状态: ✅ 已完成

## 1. 项目引导 (Bootstrap)

### 1.1 核心概念

项目引导 (Bootstrap) 是 OpenCode 初始化项目环境的核心流程，负责加载和初始化项目所需的所有服务。

**文件**: `src/project/bootstrap.ts` (72 行)

### 1.2 引导流程

```
项目启动
    │
    ▼
InstanceStore.load(input)
    │
    ▼
project.fromDirectory() - 获取项目信息
    │
    ▼
bootstrap.run() - 执行引导
    │
    ├── config.get() - 加载配置
    ├── plugin.init() - 初始化插件
    ├── lsp.init() - 初始化 LSP
    ├── format.init() - 初始化格式化
    ├── file.init() - 初始化文件服务
    ├── fileWatcher.init() - 初始化文件监控
    ├── vcs.init() - 初始化版本控制
    ├── snapshot.init() - 初始化快照
    └── project.init() - 初始化项目服务
```

### 1.3 引导服务接口

```typescript
// src/project/bootstrap-service.ts
export interface Interface {
  readonly run: Effect.Effect<void>
}

export class Service extends Context.Service<Service, Interface>()("@opencode/InstanceBootstrap") {}
```

### 1.4 并发初始化

引导过程使用 `Effect.forkScoped` 并发初始化各服务:

```typescript
yield* Effect.forEach(
  [lsp, shareNext, format, file, fileWatcher, vcs, snapshot, project],
  (s) => s.init().pipe(Effect.catchCause((cause) => Effect.logWarning("init failed", { cause }))),
  { concurrency: "unbounded", discard: true },
)
```

---

## 2. 工作目录 (Worktree)

### 2.1 核心概念

Worktree 是 Git 工作树 (git-worktree) 的封装，允许用户创建隔离的开发环境。

**文件**: `src/worktree/index.ts` (596 行)

### 2.2 Worktree 数据结构

```typescript
export const Info = Schema.Struct({
  name: Schema.String,        // 工作树名称
  branch: Schema.String,      // 分支名称
  directory: Schema.String,  // 工作树目录路径
})
```

### 2.3 工作流程

```
用户请求创建 Worktree
    │
    ▼
Worktree.makeWorktreeInfo(name?)
    │
    ├── 检查项目是否为 Git 项目
    ├── 生成唯一名称 (slug)
    └── 创建分支名 (opencode/<slug>)
    │
    ▼
Worktree.createFromInfo(info)
    │
    ├── setup: git worktree add --no-checkout -b <branch> <directory>
    └── boot: git reset --hard + 初始化项目实例
    │
    ▼
运行启动脚本 (start command)
```

### 2.4 Worktree 操作

| 操作 | 说明 |
|------|------|
| **create** | 创建新的 Worktree |
| **createFromInfo** | 根据已有信息创建 Worktree |
| **remove** | 移除 Worktree (包括删除分支) |
| **reset** | 重置 Worktree 到默认分支 |

### 2.5 Worktree 存储

- 存储位置: `~/.opencode/data/worktree/<projectID>/`
- 使用 Git worktree 机制
- 支持启动脚本 (start command)

### 2.6 Reset 流程

```
Worktree.reset(input)
    │
    ├── 获取默认分支
    ├── git fetch (如需要)
    ├── git reset --hard <default-branch>
    ├── git clean -ffdx (清理未跟踪文件)
    ├── git submodule update --init --recursive
    ├── git submodule foreach --recursive reset/clean
    └── 运行启动脚本
```

---

## 3. 会话存储 (Storage/Session)

### 3.1 数据库架构

OpenCode 使用 SQLite (通过 Drizzle ORM) 存储数据。

**文件**: `src/storage/db.ts` (182 行)

### 3.2 数据库配置

```typescript
// 数据库路径
export function getChannelPath() {
  if (["latest", "beta", "prod"].includes(InstallationChannel))
    return path.join(Global.Path.data, "opencode.db")
  return path.join(Global.Path.data, `opencode-${safe}.db`)
}

// PRAGMA 配置
db.run("PRAGMA journal_mode = WAL")
db.run("PRAGMA synchronous = NORMAL")
db.run("PRAGMA busy_timeout = 5000")
db.run("PRAGMA cache_size = -64000")
db.run("PRAGMA foreign_keys = ON")
```

### 3.3 会话表结构

**文件**: `src/session/session.sql.ts` (131 行)

#### SessionTable

```typescript
export const SessionTable = sqliteTable("session", {
  id: text().$type<SessionID>().primaryKey(),
  project_id: text().$type<ProjectID>().notNull().references(() => ProjectTable.id, { onDelete: "cascade" }),
  workspace_id: text().$type<WorkspaceID>(),
  parent_id: text().$type<SessionID>(),
  slug: text().notNull(),
  directory: text().notNull(),
  path: text(),
  title: text().notNull(),
  version: text().notNull(),
  share_url: text(),
  summary_additions: integer(),
  summary_deletions: integer(),
  summary_files: integer(),
  summary_diffs: text({ mode: "json" }).$type<Snapshot.FileDiff[]>(),
  revert: text({ mode: "json" }).$type<RevertData>(),
  permission: text({ mode: "json" }).$type<Permission.Ruleset>(),
  agent: text(),
  model: text({ mode: "json" }).$type<ModelConfig>(),
  time_compacting: integer(),
  time_archived: integer(),
  ...Timestamps,
})
```

#### MessageTable

```typescript
export const MessageTable = sqliteTable("message", {
  id: text().$type<MessageID>().primaryKey(),
  session_id: text().$type<SessionID>().notNull().references(() => SessionTable.id, { onDelete: "cascade" }),
  data: text({ mode: "json" }).notNull().$type<InfoData>(),
  ...Timestamps,
})
```

#### PartTable

```typescript
export const PartTable = sqliteTable("part", {
  id: text().$type<PartID>().primaryKey(),
  message_id: text().$type<MessageID>().notNull().references(() => MessageTable.id, { onDelete: "cascade" }),
  session_id: text().$type<SessionID>().notNull(),
  data: text({ mode: "json" }).notNull().$type<PartData>(),
  ...Timestamps,
})
```

#### TodoTable

```typescript
export const TodoTable = sqliteTable("todo", {
  session_id: text().$type<SessionID>().notNull().references(() => SessionTable.id, { onDelete: "cascade" }),
  content: text().notNull(),
  status: text().notNull(),
  priority: text().notNull(),
  position: integer().notNull(),
  ...Timestamps,
})
```

#### SessionMessageTable (V2 消息)

```typescript
export const SessionMessageTable = sqliteTable("session_message", {
  id: text().$type<SessionMessage.ID>().primaryKey(),
  session_id: text().$type<SessionID>().notNull().references(() => SessionTable.id, { onDelete: "cascade" }),
  type: text().$type<SessionMessage.Type>().notNull(),
  data: text({ mode: "json" }).notNull().$type<SessionMessageData>(),
  ...Timestamps,
})
```

### 3.4 事务和上下文

```typescript
// 局部上下文 (Local Context)
const ctx = LocalContext.create<{
  tx: TxOrDb
  effects: (() => void | Promise<void>)[]
}>("database")

// 使用数据库
export function use<T>(callback: (trx: TxOrDb) => T): T {
  try {
    return callback(ctx.use().tx)
  } catch (err) {
    if (err instanceof LocalContext.NotFound) {
      // 创建新上下文
      const result = ctx.provide({ effects: [], tx: Client() }, () => callback(Client()))
      return result
    }
    throw err
  }
}

// 事务支持
export function transaction<T>(callback: (tx: TxOrDb) => T): T {
  try {
    return callback(ctx.use().tx)
  } catch (err) {
    if (err instanceof LocalContext.NotFound) {
      // 创建事务
      return Client().transaction(txCallback, { behavior: "deferred" })
    }
    throw err
  }
}
```

### 3.5 项目数据

**文件**: `src/project/project.sql.ts`

```typescript
export const ProjectTable = sqliteTable("project", {
  id: text().$type<ProjectID>().primaryKey(),
  worktree: text().notNull(),
  sandbox: text().notNull(),
  vcs: text(),
  name: text(),
  icon_url: text(),
  icon_url_override: text(),
  icon_color: text(),
  commands: text({ mode: "json" }).$type<ProjectCommands>(),
  time_created: integer().notNull(),
  time_updated: integer().notNull(),
  time_initialized: integer(),
})
```

---

## 4. 实例管理 (Instance)

### 4.1 实例上下文

```typescript
// src/project/instance-context.ts
export interface InstanceContext {
  readonly directory: string      // 工作目录
  readonly worktree: string        // Git worktree 路径
  readonly project: Project.Info  // 项目信息
}
```

### 4.2 实例存储

**文件**: `src/project/instance-store.ts` (191 行)

```typescript
export interface Interface {
  readonly load: (input: LoadInput) => Effect.Effect<InstanceContext>
  readonly reload: (input: LoadInput) => Effect.Effect<InstanceContext>
  readonly dispose: (ctx: InstanceContext) => Effect.Effect<void>
  readonly disposeAll: () => Effect.Effect<void>
  readonly provide: <A, E, R>(input: LoadInput, effect: Effect.Effect<A, E, R>) => Effect.Effect<A, E, R>
}
```

### 4.3 实例加载流程

```typescript
const boot = (input: LoadInput & { directory: string }) =>
  Effect.gen(function* () {
    const ctx: InstanceContext = yield* project.fromDirectory(input.directory).pipe(
      Effect.map((result) => ({
        directory: input.directory,
        worktree: result.sandbox,
        project: result.project,
      })),
    )
    yield* bootstrap.run.pipe(Effect.provideService(InstanceRef, ctx))
    return ctx
  })
```

### 4.4 实例生命周期

```
load() → 创建实例 → 返回 InstanceContext
    │
    ├── bootstrap.run() → 初始化所有服务
    │
    └── dispose() → 清理资源 → 触发事件
```

---

## 5. 关键文件索引

| 文件 | 行数 | 说明 |
|------|------|------|
| `src/project/bootstrap.ts` | 72 | 项目引导实现 |
| `src/project/bootstrap-service.ts` | 9 | 引导服务接口 |
| `src/project/instance-store.ts` | 191 | 实例存储管理 |
| `src/project/instance-layer.ts` | 11 | 实例层组合 |
| `src/project/instance-context.ts` | - | 实例上下文类型 |
| `src/project/project.ts` | 540 | 项目服务主文件 |
| `src/project/project.sql.ts` | - | 项目数据库 Schema |
| `src/worktree/index.ts` | 596 | Worktree 服务 |
| `src/storage/db.ts` | 182 | 数据库连接管理 |
| `src/storage/schema.sql.ts` | - | 存储 Schema |
| `src/session/session.sql.ts` | 131 | 会话数据库 Schema |

---

## 6. 系统交互图

```
┌─────────────────────────────────────────────────────────────┐
│                      入口点                                 │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────────────┐   │
│  │ CLI 启动    │  │ Server 启动  │  │ ACP 连接         │   │
│  └──────┬──────┘  └──────┬──────┘  └────────┬─────────┘   │
└─────────┼────────────────┼───────────────────┼─────────────┘
          │                │                   │
          ▼                ▼                   ▼
┌─────────────────────────────────────────────────────────────┐
│                   InstanceStore.load()                      │
│                                                              │
│  ┌─────────────────────────────────────────────────────────┐│
│  │ 1. project.fromDirectory() → 获取 Project.Info          ││
│  │ 2. bootstrap.run() → 初始化所有服务                     ││
│  └─────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Bootstrap.run()                           │
│                                                              │
│  config.get() → plugin.init() → lsp.init() → format.init()  │
│  → file.init() → fileWatcher.init() → vcs.init()           │
│  → snapshot.init() → project.init()                        │
└─────────────────────────────────────────────────────────────┘
                              │
          ┌───────────────────┼───────────────────┐
          ▼                   ▼                   ▼
┌─────────────────┐  ┌───────────────┐  ┌────────────────┐
│   数据库        │  │ 文件系统      │  │  Git 服务      │
│                 │  │               │  │                │
│ - SessionTable  │  │ - 读取        │  │ - git status   │
│ - MessageTable  │  │ - 写入        │  │ - git diff     │
│ - PartTable     │  │ - 监控        │  │ - worktree     │
│ - ProjectTable  │  │               │  │                │
└─────────────────┘  └───────────────┘  └────────────────┘
```

---

## 7. 待深入研究的问题

1. **项目发现机制**: 项目如何被自动发现?
2. **实例隔离**: 不同实例之间如何隔离?
3. **会话压缩**: 会话数据如何压缩和归档?
4. **启动脚本**: Worktree 启动脚本的具体执行逻辑?

---

## 8. 总结

Phase 10 完成了对项目和会话管理系统的深入理解:

- ✅ 理解了项目引导流程 (Bootstrap)
- ✅ 理解了 Worktree 机制 (Git worktree 封装)
- ✅ 理解了会话存储 (SQLite/Drizzle)
- ✅ 理解了实例管理 (Instance Store/Context)

**关键发现**:
- Bootstrap 使用并发初始化多个服务
- Worktree 基于 Git worktree，提供隔离开发环境
- 使用 Drizzle ORM + SQLite 存储会话数据
- 支持 V2 消息系统 (SessionMessageTable)
- 实例存储使用 Map 缓存，支持加载/卸载

**下一步**: Phase 11 - 基础设施服务