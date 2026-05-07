# 插件系统研究报告

> 研究时间: ~2 小时
> 状态: ✅ 已完成
> 关联 Phase: 8

---

## 1. 插件类型

### 1.1 内置插件

直接导入，不通过 npm 安装:

```typescript
const INTERNAL_PLUGINS: PluginInstance[] = [
  CodexAuthPlugin,           // Codex 认证
  CopilotAuthPlugin,        // GitHub Copilot 认证
  GitlabAuthPlugin,         // GitLab 认证
  PoeAuthPlugin,           // Poe 认证
  CloudflareWorkersAuthPlugin,  // Cloudflare Workers
  CloudflareAIGatewayAuthPlugin, // Cloudflare AI Gateway
  AzureAuthPlugin,         // Azure 认证
]
```

### 1.2 外部插件

通过 npm 安装，配置在 `opencode.json` 中:

```typescript
plugin: [
  { npm: "@opencode-ai/plugin-example" },
  { path: "./my-plugin" },
  "plugin-name"  // npm 包名
]
```

---

## 2. 插件接口

### 2.1 插件输入

```typescript
interface PluginInput {
  client: OpencodeClient      // SDK 客户端
  project: Project            // 项目信息
  worktree: string            // 工作树根目录
  directory: string          // 当前目录
  experimental_workspace: {
    register(type: string, adapter: WorkspaceAdapter): void
  }
  serverUrl: URL             // 服务器 URL
  $?: Bun                   // Bun 运行时 (可选)
}
```

### 2.2 插件 Hooks

```typescript
interface Hooks {
  // 模型相关
  "model.transform"?: (input, output) => Promise<void>
  "model.variants"?: (input, output) => Promise<void>
  
  // 消息相关
  "chat.messages.transform"?: (input, output) => Promise<void>
  
  // 压缩相关
  "session.compacting"?: (input, output) => Promise<void>
  "compaction.autocontinue"?: (input, output) => Promise<void>
  
  // Provider 相关
  "provider.auth.refresh"?: (input, output) => Promise<void>
  "provider.create"?: (input, output) => Promise<void>
  
  // 工具相关
  "tool.transform"?: (input, output) => Promise<void>
  "tool.permission.ask"?: (input, output) => Promise<void>
  
  // 生命周期
  "server.started"?: (input, output) => Promise<void>
  "server.stopping"?: (input, output) => Promise<void>
  "project.started"?: (input, output) => Promise<void>
  "project.stopping"?: (input, output) => Promise<void>
}
```

---

## 3. 插件服务

### 3.1 服务接口

```typescript
interface Interface {
  // 触发 Hook
  trigger<
    Name extends TriggerName,
    Input = Parameters<Required<Hooks>[Name]>[0],
    Output = Parameters<Required<Hooks>[Name]>[1],
  >(name: Name, input: Input, output: Output): Effect.Effect<Output>
  
  // 列出所有插件 Hooks
  list(): Effect.Effect<Hooks[]>
  
  // 初始化插件
  init(): Effect.Effect<void>
}
```

### 3.2 Hook 触发

```typescript
// 遍历所有插件，执行同名 Hook
trigger(name: Name, input: Input, output: Output): Effect.Effect<Output> {
  return Effect.gen(function* () {
    const hooks = yield* list()
    let result = output
    for (const hook of hooks) {
      const fn = hook[name]
      if (fn) {
        yield* Effect.tryPromise(() => fn(input, result))
      }
    }
    return result
  })
}
```

---

## 4. 插件加载

### 4.1 加载流程

1. 加载内置插件
2. 等待依赖安装完成
3. 加载外部插件 (通过 PluginLoader)
4. 安装 npm 包 (如需要)
5. 解析入口点并初始化

### 4.2 插件加载器

```typescript
// PluginLoader.loadExternal 负责:
// 1. 解析插件规格
// 2. 安装 npm 包 (如需要)
// 3. 加载模块
// 4. 调用入口函数
```

### 4.3 版本支持

- V1 插件: 使用 `readV1Plugin` 检测
- 遗留插件: 使用 `getLegacyPlugins` 处理

---

## 5. 关键文件索引

| 文件 | 说明 |
|------|------|
| `src/plugin/index.ts` | 插件服务 (288 行) |
| `src/plugin/loader.ts` | 插件加载器 |
| `src/plugin/install.ts` | 插件安装 |
| `src/plugin/shared.ts` | 共享工具 |
| `src/plugin/codex.ts` | Codex 认证插件 |
| `src/plugin/cloudflare.ts` | Cloudflare 插件 |
| `src/plugin/azure.ts` | Azure 插件 |

---

## 6. 总结

- **两种插件**: 内置 (直接导入) + 外部 (npm 安装)
- **Hook 机制**: 基于事件的可扩展点
- **动态加载**: 支持 npm 包和本地路径
- **认证集成**: 多个 OAuth 认证插件