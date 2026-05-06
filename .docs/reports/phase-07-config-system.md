# 配置系统研究报告

---

## 1. 配置文件

### 1.1 配置文件位置

配置文件: `opencode.json` (项目根目录)

### 1.2 配置结构

```typescript
interface ConfigInfo {
  // 基础配置
  $schema?: string           // JSON schema 引用
  shell?: string            // 默认 shell
  logLevel?: "DEBUG" | "INFO" | "WARN" | "ERROR"
  
  // 服务器配置
  server?: Server
  
  // 命令配置
  command?: Record<string, CommandConfig>
  
  // Skills 配置
  skills?: Skills
  
  // 文件监控
  watcher?: { ignore?: string[] }
  
  // 快照
  snapshot?: boolean
  
  // 插件
  plugin?: Plugin[]
  plugin_origins?: Plugin.Origin[]  // 内部使用的来源追踪
  
  // 分享
  share?: "manual" | "auto" | "disabled"
  autoshare?: boolean
  autoupdate?: boolean | "notify"
  
  // Provider 配置
  disabled_providers?: string[]
  enabled_providers?: string[]
  
  // 模型配置
  model?: "provider/model"      // 默认模型
  small_model?: "provider/model" // 小模型 (用于 title 生成)
  default_agent?: string         // 默认 Agent
  username?: string             // 自定义用户名
  
  // Agent 配置
  agent?: {
    // Primary agents
    plan?: AgentConfig
    build?: AgentConfig
    // Subagents
    general?: AgentConfig
    explore?: AgentConfig
    // Specialized
    title?: AgentConfig
    summary?: AgentConfig
    compaction?: AgentConfig
    // Custom agents
    [key: string]: AgentConfig
  }
  
  // 自定义 Provider
  provider?: Record<string, ProviderConfig>
  
  // MCP 配置
  mcp?: Record<string, MCPConfig | { enabled: false }>
  
  // Formatter 配置
  formatter?: FormatterConfig
  
  // LSP 配置
  lsp?: LSPConfig
  
  // 指令文件
  instructions?: string[]
  
  // 权限配置
  permission?: PermissionConfig
  
  // 工具配置
  tools?: Record<string, boolean>
  
  // 企业配置
  enterprise?: { url?: string }
  
  // 工具输出限制
  tool_output?: {
    max_lines?: number
    max_bytes?: number
  }
  
  // 压缩配置
  compaction?: {
    auto?: boolean
    prune?: boolean
    tail_turns?: number
    preserve_recent_tokens?: number
    reserved?: number
  }
  
  // 实验性功能
  experimental?: {
    disable_paste_summary?: boolean
    batch_tool?: boolean
    openTelemetry?: boolean
    primary_tools?: string[]
    continue_loop_on_deny?: boolean
    mcp_timeout?: number
  }
}
```

---

## 2. 配置模块

### 2.1 配置模块列表

| 模块 | 文件 | 说明 |
|------|------|------|
| agent | `config/agent.ts` | Agent 配置 |
| command | `config/command.ts` | 命令配置 |
| formatter | `config/formatter.ts` | 格式化器 |
| layout | `config/layout.ts` | 布局配置 |
| lsp | `config/lsp.ts` | LSP 配置 |
| managed | `config/managed.ts` | 托管配置 |
| mcp | `config/mcp.ts` | MCP 配置 |
| model-id | `config/model-id.ts` | 模型 ID |
| parse | `config/parse.ts` | 解析配置 |
| paths | `config/paths.ts` | 路径配置 |
| permission | `config/permission.ts` | 权限配置 |
| plugin | `config/plugin.ts` | 插件配置 |
| provider | `config/provider.ts` | Provider 配置 |
| server | `config/server.ts` | 服务器配置 |
| skills | `config/skills.ts` | Skills 配置 |
| variable | `config/variable.ts` | 变量配置 |

---

## 3. 配置加载

### 3.1 配置优先级

配置按以下顺序合并 (后者覆盖前者):
1. 全局配置 (`~/.config/opencode/opencode.json`)
2. 项目配置 (`./opencode.json`)
3. 环境变量覆盖

### 3.2 配置合并

```typescript
// mergeConfig 使用 remeda 的 deep merge
function mergeConfig(target: Info, source: Info): Info {
  return mergeDeep(target, source) as Info
}

// instructions 数组会合并而非替换
if (target.instructions && source.instructions) {
  merged.instructions = Array.from(new Set([...target.instructions, ...source.instructions]))
}
```

### 3.3 配置服务接口

```typescript
interface Interface {
  // 获取当前目录配置
  get(): Effect.Effect<Info>
  
  // 获取全局配置
  getGlobal(): Effect.Effect<Info>
  
  // 获取控制台状态
  getConsoleState(): Effect.Effect<ConsoleState>
  
  // 更新当前配置
  update(config: Info): Effect.Effect<void>
  
  // 更新全局配置
  updateGlobal(config: Info): Effect.Effect<{ info: Info; changed: boolean }>
  
  // 使配置失效 (重新加载)
  invalidate(): Effect.Effect<void>
  
  // 获取配置目录列表
  directories(): Effect.Effect<string[]>
  
  // 等待依赖安装
  waitForDependencies(): Effect.Effect<void>
}
```

---

## 4. 关键文件

| 文件 | 说明 |
|------|------|
| `src/config/config.ts` | 主配置文件 (788 行) |
| `src/config/agent.ts` | Agent 配置 Schema |
| `src/config/mcp.ts` | MCP 配置 Schema (65 行) |
| `src/config/provider.ts` | Provider 配置 Schema |
| `src/config/plugin.ts` | 插件配置 Schema |

---

## 5. 总结

- **配置文件**: JSON 格式 (`opencode.json`)
- **配置合并**: 深度合并，数组去重
- **模块化**: 20+ 子模块按功能分类
- **支持**: Provider/MCP/Agent/Plugin/LSP 等配置