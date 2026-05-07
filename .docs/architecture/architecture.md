# Phase 1: 基础架构 - 研究报告

> 研究时间: ~2 小时
> 状态: ✅ 已完成

## 1. 项目结构理解

### 1.1 Monorepo 架构

OpenCode 是一个基于 **Turborepo** 的 monorepo 项目，使用 **Bun** 作为运行时和包管理器。

```
根目录配置:
- package.json: workspace 定义，catalog 共享依赖版本
- turbo.json: 构建任务配置 (typecheck, build, test)
- packages/: 20+ 个子包
```

### 1.2 主要包及职责

| 包名 | 职责 | 关键文件 |
|------|------|----------|
| `opencode` | 核心 CLI 应用程序，包含所有业务逻辑 | `src/index.ts` |
| `@opencode-ai/core` | 核心工具库 (日志、文件系统、Hash、Glob 等) | `src/util/log.ts` |
| `app` | 前端 Web 应用 (SolidJS) | `src/pages/` |
| `ui` | UI 组件库 | - |
| `console` | 控制台 UI | - |
| `web` | 官方网站 | - |
| `docs` | 文档 | - |
| `sdk` | SDK (JS/Python) | `packages/sdk/js/`, `packages/sdk/python/` |
| `plugin` | 插件系统 | `src/plugin/` |
| `extensions` | 扩展 | - |

### 1.3 依赖关系

```
@opencode-ai/core (基础工具)
    ↓ 依赖
opencode (核心业务)
    ↓ 依赖
app/ui/console (前端)
```

### 1.4 关键配置文件

**package.json** (根目录)
- workspace 定义: `packages/*`, `packages/console/*`, `packages/sdk/js`, `packages/slack`
- catalog 共享依赖版本管理
- scripts: `dev`, `dev:web`, `dev:console`, `typecheck`, `test`

**turbo.json**
- `typecheck`: 全局类型检查
- `build`: 构建 (输出 `dist/**`)
- 测试任务配置

---

## 2. 入口点和启动流程

### 2.1 CLI 入口点

**文件**: `packages/opencode/src/index.ts` (247 行)

```typescript
// 使用 yargs 解析命令行参数
const cli = yargs(args)
  .scriptName("opencode")
  .wrap(100)
  .help("help", "show help")
  .version("version", "show version", InstallationVersion)
  .option("print-logs", { describe: "print logs to stderr", type: "boolean" })
  .option("log-level", { describe: "log level", choices: ["DEBUG", "INFO", "WARN", "ERROR"] })
  .option("pure", { describe: "run without external plugins", type: "boolean" })
  .middleware(async (opts) => {
    // 初始化日志
    // 数据库迁移检查
  })
  .command(AcpCommand)
  .command(McpCommand)
  .command(RunCommand)
  // ... 共 20+ 命令
```

### 2.2 命令列表

| 命令 | 说明 | 文件 |
|------|------|------|
| `run` | 启动 Agent 会话 (主要) | `src/cli/cmd/run.ts` |
| `serve` | 启动 HTTP Server | `src/cli/cmd/serve.ts` |
| `acp` | 启动 ACP 服务端 | `src/cli/cmd/acp.ts` |
| `mcp` | MCP 服务器管理 | `src/cli/cmd/mcp.ts` |
| `models` | 模型列表 | `src/cli/cmd/models.ts` |
| `providers` | Provider 管理 | `src/cli/cmd/providers.ts` |
| `account` | 账户管理 | `src/cli/cmd/account.ts` |
| `db` | 数据库操作 | `src/cli/cmd/db.ts` |
| `tui` | 启动 TUI 界面 | - |
| `attach` | 附加到远程会话 | - |

### 2.3 启动流程

```
1. index.ts 入口
   ↓
2. yargs 解析参数
   ↓
3. 中间件 (Middleware):
   - 日志初始化 (Log.init)
   - 数据库迁移检查 (JsonMigration.run)
   - Heap 性能监控启动
   - 环境变量设置 (AGENT, OPENCODE, OPENCODE_PID)
   ↓
4. 执行对应命令 (Command Handler)
   ↓
5. effect-cmd 处理 Effect 上下文
   ↓
6. AppRuntime.runPromise() 执行
```

### 2.4 effect-cmd 机制

**文件**: `packages/opencode/src/cli/effect-cmd.ts` (103 行)

```typescript
interface EffectCmdOpts<Args, A> {
  command: string | readonly string[]
  instance?: boolean | ((args: Args) => boolean)  // 是否需要项目实例
  directory?: (args: Args) => string              // 工作目录
  handler: (args) => Effect.Effect<A>             // Effect 处理函数
}

// 两种模式:
// 1. instance=true (默认): 需要加载项目上下文
await AppRuntime.runPromise(
  InstanceStore.Service.use(store => store.load({ directory }))
)
// -> Instance.restore(ctx, () => AppRuntime.runPromise(handler))

// 2. instance=false: 直接运行 (e.g., models, serve)
await AppRuntime.runPromise(opts.handler(args))
```

---

## 3. Effect 框架基础

### 3.1 Effect 核心概念

项目使用 **Effect** (v4 beta) 作为函数式编程框架，实现依赖注入和服务管理。

#### Service 定义模式

**文件**: `AGENTS.md` (项目规范)

```typescript
// 文件: src/foo/foo.ts

// 1. 定义接口
export interface Interface {
  readonly method: () => Effect.Effect<string>
}

// 2. 定义 Service
export class Service extends Context.Service<Service, Interface>()("@opencode/Foo") {}

// 3. 创建 Layer
export const layer = Layer.effect(
  Service,
  Effect.gen(function* () {
    return Service.of({
      method: () => Effect.succeed("result")
    })
  })
)

// 4. 派生默认 Layer
export const defaultLayer = layer.pipe(
  Layer.provide(OtherService.defaultLayer)
)

// 5. 自导出
export * as Foo from "./foo"
```

#### 使用 Service

```typescript
import { Foo } from "@/foo/foo"

yield* Foo.Service  // 获取服务实例
Foo.layer           // Layer 用于组合
```

### 3.2 Effect 命名规范

| 函数 | 用途 | 示例 |
|------|------|------|
| `Effect.fn("Domain.method")` | 命名可追踪的 Effect | `Effect.fn("Session.run")(function*() {...})` |
| `Effect.fnUntraced` | 内部辅助函数 (不追踪) | `Effect.fnUntraced(function*() {...})` |
| `Effect.gen(function* () {})` | 组合多个 Effect | `Effect.gen(function*() { const a = yield* ... })` |
| `Effect.callback` | 回调式 API 封装 | `Effect.callback((resolve) => { ... })` |

### 3.3 运行时模式

**文件**: `packages/opencode/src/effect/app-runtime.ts` (133 行)

```typescript
// 全局 Layer 组合 - 40+ 服务
export const AppLayer = Layer.mergeAll(
  Npm.defaultLayer,
  AppFileSystem.defaultLayer,
  Bus.defaultLayer,
  Auth.defaultLayer,
  Account.defaultLayer,
  Config.defaultLayer,
  // ... 更多服务
)

// 运行 Effect
AppRuntime.runPromise(effect)
AppRuntime.runFork(effect)   // 后台执行
AppRuntime.runCallback(effect) // 回调风格
```

### 3.4 Runtime vs InstanceState

| 类型 | 用途 | 文件 |
|------|------|------|
| `makeRuntime` | 全局共享服务 (通过 memoMap 去重) | `src/effect/run-service.ts` |
| `InstanceState` | 每项目独立状态 (ScopedCache 按目录隔离) | `src/effect/instance-state.ts` |

**InstanceState 使用场景**:
- 每个打开的项目需要独立状态
- 状态需要在项目关闭时清理
- 使用 `Effect.addFinalizer` 进行资源清理

---

## 4. 关键文件索引

| 分类 | 文件 | 行数 | 说明 |
|------|------|------|------|
| **入口** | `src/index.ts` | 247 | CLI 入口，命令注册，中间件 |
| 命令基础设施 | `src/cli/effect-cmd.ts` | 103 | Effect 命令封装 |
| 启动命令示例 | `src/cli/cmd/run.ts` | 678 | run 命令实现 |
| 运行时 | `src/effect/app-runtime.ts` | 133 | AppLayer 定义 |
| Effect 规范 | `AGENTS.md` | 137 | 项目规范文档 |

---

## 5. 待深入研究的问题

1. **中间件执行顺序**: 数据库迁移在每次启动时检查还是仅首次?
2. **Instance 生命周期**: InstanceStore 的加载和清理机制
3. **命令扩展**: 如何注册新的 CLI 命令
4. **配置加载时机**: Config 服务在启动何时加载?

---

## 6. 总结

Phase 1 完成了对 OpenCode 项目基础架构的理解:

- ✅ 理解了 Monorepo 结构和主要包的职责
- ✅ 理解了 CLI 入口点和命令注册机制
- ✅ 理解了 Effect 框架的核心概念和使用模式
- ✅ 理解了启动流程和中间件机制

**下一步**: Phase 2 - Agent Loop (会话循环)