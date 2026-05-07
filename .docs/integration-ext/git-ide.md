# Phase 12: Git 和 IDE 集成 - 研究报告

> 研究时间: ~2 小时
> 状态: ✅ 已完成

## 1. Git 集成

### 1.1 核心概念

Git 服务封装了常用的 Git 操作，支持工作目录管理、差异计算等。

**文件**: `src/git/index.ts` (352 行)

### 1.2 Git 配置

```typescript
const cfg = [
  "--no-optional-locks",
  "-c", "core.autocrlf=false",
  "-c", "core.fsmonitor=false",
  "-c", "core.longpaths=true",
  "-c", "core.symlinks=true",
  "-c", "core.quotepath=false",
] as const
```

### 1.3 接口定义

```typescript
export interface Interface {
  readonly run: (args: string[], opts: Options) => Effect.Effect<Result>
  readonly branch: (cwd: string) => Effect.Effect<string | undefined>
  readonly prefix: (cwd: string) => Effect.Effect<string>
  readonly defaultBranch: (cwd: string) => Effect.Effect<Base | undefined>
  readonly hasHead: (cwd: string) => Effect.Effect<boolean>
  readonly mergeBase: (cwd: string, base: string, head?: string) => Effect.Effect<string | undefined>
  readonly show: (cwd: string, ref: string, file: string, prefix?: string) => Effect.Effect<string>
  readonly status: (cwd: string) => Effect.Effect<Item[]>
  readonly diff: (cwd: string, ref: string) => Effect.Effect<Item[]>
  readonly stats: (cwd: string, ref: string) => Effect.Effect<Stat[]>
  readonly patch: (cwd: string, ref: string, file: string, options?: PatchOptions) => Effect.Effect<Patch>
  readonly patchAll: (cwd: string, ref: string, options?: PatchOptions) => Effect.Effect<Patch>
  readonly patchUntracked: (cwd: string, file: string, options?: PatchOptions) => Effect.Effect<Patch>
  readonly statUntracked: (cwd: string, file: string) => Effect.Effect<Stat | undefined>
}
```

### 1.4 常用操作

#### 获取当前分支

```typescript
const branch = Effect.fn("Git.branch")(function* (cwd: string) {
  const result = yield* run(["symbolic-ref", "--quiet", "--short", "HEAD"], { cwd })
  if (result.exitCode !== 0) return
  return out(result) || undefined
})
```

#### 获取默认分支

```typescript
const defaultBranch = Effect.fn("Git.defaultBranch")(function* (cwd: string) {
  const remote = yield* primary(cwd)
  if (remote) {
    const head = yield* run(["symbolic-ref", `refs/remotes/${remote}/HEAD`], { cwd })
    if (head.exitCode === 0) {
      const ref = out(head).replace(/^refs\/remotes\//, "")
      const name = ref.startsWith(`${remote}/`) ? ref.slice(`${remote}/`.length) : ""
      if (name) return { name, ref }
    }
  }
  // Fallback: 检查本地配置或常见分支名
})
```

#### 获取文件状态

```typescript
const status = Effect.fn("Git.status")(function* (cwd: string) {
  return nuls(
    yield* text(["status", "--porcelain=v1", "--untracked-files=all", "--no-renames", "-z", "--", "."], {
      cwd,
    }),
  ).flatMap((item) => {
    const file = item.slice(3)
    if (!file) return []
    const code = item.slice(0, 2)
    return [{ file, code, status: kind(code) }]
  })
})
```

#### 获取差异统计

```typescript
const stats = Effect.fn("Git.stats")(function* (cwd: string, ref: string) {
  return nuls(
    yield* text(["diff", "--no-ext-diff", "--no-renames", "--numstat", "-z", ref, "--", "."], { cwd }),
  ).flatMap((item) => {
    // 解析添加/删除行数
    const parts = item.split("\t")
    const file = parts[2]
    const additions = Number.parseInt(parts[0] || "0", 10)
    const deletions = Number.parseInt(parts[1] || "0", 10)
    return [{ file, additions, deletions }]
  })
})
```

#### 生成补丁

```typescript
const patch = Effect.fn("Git.patch")(function* (cwd: string, ref: string, file: string, options?: PatchOptions) {
  const result = yield* run(
    ["diff", "--patch", "--no-ext-diff", "--no-renames", `--unified=${options?.context ?? 3}`, ref, "--", file],
    { cwd, maxOutputBytes: options?.maxOutputBytes },
  )
  return { text: result.truncated ? "" : result.text(), truncated: result.truncated }
})
```

### 1.5 结果类型

```typescript
export type Result = {
  readonly exitCode: number
  readonly text: () => string
  readonly stdout: Buffer
  readonly stderr: Buffer
  readonly truncated: boolean
}

export type Kind = "added" | "deleted" | "modified"

export type Item = {
  readonly file: string
  readonly code: string
  readonly status: Kind
}

export type Stat = {
  readonly file: string
  readonly additions: number
  readonly deletions: number
}

export type Patch = {
  readonly text: string
  readonly truncated: boolean
}
```

---

## 2. LSP 集成

### 2.1 核心概念

LSP (Language Server Protocol) 集成提供代码补全、诊断等功能。

**文件**: `src/lsp/lsp.ts` (517 行)

### 2.2 LSP 事件

```typescript
export const Event = {
  Updated: BusEvent.define("lsp.updated", Schema.Struct({})),
}
```

### 2.3 LSP 数据类型

```typescript
export const Range = Schema.Struct({
  start: Position,
  end: Position,
})

export const Symbol = Schema.Struct({
  name: Schema.String,
  kind: NonNegativeInt,
  location: Schema.Struct({
    uri: Schema.String,
    range: Range,
  }),
})

export const Status = Schema.Struct({
  id: Schema.String,
  name: Schema.String,
  root: Schema.String,
  status: Schema.Literals(["connected", "error"]),
})
```

### 2.4 LSP 服务结构

```
src/lsp/
├── lsp.ts        - LSP 服务主文件
├── client.ts     - LSP 客户端
├── server.ts     - LSP 服务端
├── launch.ts     - LSP 进程启动
├── language.ts   - 语言检测
└── diagnostic.ts - 诊断信息
```

### 2.5 功能概览

| 功能 | 说明 |
|------|------|
| **代码补全** | 通过 LSP 实现代码自动补全 |
| **诊断信息** | 获取语言服务器报告的错误和警告 |
| **符号查询** | 查询文件中的符号 (类、函数等) |
| **代码导航** | 跳转到定义、引用等 |

---

## 3. 工具集成

### 3.1 Patch 应用工具

**文件**: `src/tool/apply_patch.ts`

使用 Git 差异来应用修改:

```typescript
// 调用 Git.Service 进行 patch 操作
const patch = yield* git.patch(cwd, ref, file, { context: 3, maxOutputBytes: 1024 * 1024 })
```

### 3.2 差异计算工具

内置差异计算:

```typescript
// 在工具中直接使用 Git 服务
const diff = yield* git.diff(cwd, ref)
const stats = yield* git.stats(cwd, ref)
```

---

## 4. 关键文件索引

| 文件 | 行数 | 说明 |
|------|------|------|
| `src/git/index.ts` | 352 | Git 服务主文件 |
| `src/lsp/lsp.ts` | 517 | LSP 服务主文件 |
| `src/lsp/client.ts` | - | LSP 客户端 |
| `src/lsp/server.ts` | - | LSP 服务端 |
| `src/lsp/launch.ts` | - | LSP 启动器 |
| `src/lsp/language.ts` | - | 语言检测 |
| `src/lsp/diagnostic.ts` | - | 诊断信息 |
| `src/tool/apply_patch.ts` | - | Patch 应用工具 |

---

## 5. 系统交互图

```
┌─────────────────────────────────────────────────────────────┐
│                      Git 服务                               │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Git.Service                                        │  │
│  │  - branch() / defaultBranch()                       │  │
│  │  - status() / diff() / stats()                     │  │
│  │  - patch() / show()                                │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
          │               │               │
          ▼               ▼               ▼
┌─────────────────┐  ┌───────────────┐  ┌────────────────┐
│   Worktree     │  │   会话摘要    │  │   工具执行     │
│                │  │               │  │                │
│ - worktree add │  │ - diff stats  │  │ - apply_patch │
│ - worktree rm  │  │ - file change │  │ - git diff    │
└─────────────────┘  └───────────────┘  └────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                      LSP 服务                              │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  LSP.Service                                        │  │
│  │  - init() / status                                 │  │
│  │  - 补全 / 诊断 / 符号                              │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

---

## 6. 总结

Phase 12 完成了对 Git 和 IDE 集成的深入理解:

- ✅ 理解了 Git 服务封装 (常用操作 + 结果类型)
- ✅ 理解了 LSP 集成架构
- ✅ 理解了工具与 Git 的集成

**关键发现**:
- Git 服务使用 `ChildProcess` 执行命令
- 支持输出截断防止大仓库性能问题
- LSP 使用语言服务器协议进行代码分析
- Patch 工具直接使用 Git.diff 输出

**下一步**: Phase 13 - SDK 和扩展