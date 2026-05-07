# Phase 13: SDK 和扩展 - 研究报告

> 研究时间: ~1.5 小时
> 状态: ✅ 已完成

## 1. JavaScript SDK

### 1.1 位置

`packages/sdk/js/`

### 1.2 目录结构

```
packages/sdk/js/
├── src/
│   ├── index.ts           # 主入口
│   ├── client.ts          # 客户端
│   ├── server.ts          # 服务端
│   ├── process.ts         # 进程管理
│   ├── v2/                # V2 版本
│   │   ├── client.ts
│   │   ├── server.ts
│   │   ├── index.ts
│   │   ├── data.ts
│   │   └── gen/           # 生成的类型
│   └── gen/               # 生成的类型
├── script/
│   ├── build.ts          # 构建脚本
│   └── publish.ts        # 发布脚本
└── example/
    └── example.ts         # 使用示例
```

### 1.3 核心功能

```typescript
// packages/sdk/js/src/index.ts
export * from "./client.js"
export * from "./server.js"

export async function createOpencode(options?: ServerOptions) {
  const server = await createOpencodeServer({ ...options })
  const client = createOpencodeClient({ baseUrl: server.url })
  return { client, server }
}
```

### 1.4 客户端 API

```typescript
// 创建客户端
const client = createOpencodeClient({ baseUrl: "http://localhost:3000" })

// 使用客户端
const sessions = await client.v2.session.list()
const messages = await client.v2.session.messages({ sessionId: "xxx" })
```

### 1.5 V2 版本

V2 SDK 提供更完整的 API:

```typescript
// V2 客户端
import { createOpencodeClient } from "./v2/client"

const client = createOpencodeClient({
  baseUrl: "http://localhost:3000",
  // 认证
  apiKey: "your-api-key",
})
```

### 1.6 构建脚本

```typescript
// packages/sdk/js/script/build.ts
// 生成 SDK 类型和客户端代码
// 运行方式: ./packages/sdk/js/script/build.ts
```

---

## 2. Python SDK

### 2.1 位置

`packages/sdk/python/`

### 2.2 概述

Python SDK 提供与 JavaScript SDK 类似的功能:

- 连接到 OpenCode 服务器
- 管理会话和消息
- 调用工具

---

## 3. 共享类型

### 3.1 会话共享

**文件**: `src/share/session.ts`

```typescript
// 跨 SDK 共享的会话类型定义
```

---

## 4. 扩展机制

### 4.1 插件系统

参考 Phase 8 - 插件系统。

### 4.2 MCP 扩展

参考 Phase 5 - MCP 集成。

---

## 5. 关键文件索引

| 文件 | 说明 |
|------|------|
| `packages/sdk/js/src/index.ts` | JS SDK 主入口 |
| `packages/sdk/js/src/client.ts` | 客户端实现 |
| `packages/sdk/js/src/server.ts` | 服务端实现 |
| `packages/sdk/js/src/v2/client.ts` | V2 客户端 |
| `packages/sdk/js/script/build.ts` | 构建脚本 |
| `packages/sdk/python/` | Python SDK 目录 |
| `src/share/session.ts` | 共享类型 |

---

## 6. 使用示例

```typescript
// JavaScript SDK 示例
import { createOpencode } from "@opencode-ai/sdk"

async function main() {
  // 创建服务器和客户端
  const { client, server } = await createOpencode({
    // 选项
  })

  // 列出会话
  const sessions = await client.session.list()

  // 发送消息
  const response = await client.session.send({
    sessionId: "xxx",
    message: "Hello!",
  })

  // 清理
  await server.close()
}
```

---

## 7. 总结

Phase 13 完成了对 SDK 和扩展的深入理解:

- ✅ 理解了 JavaScript SDK 结构
- ✅ 理解了 Python SDK
- ✅ 理解了 V2 API

**关键发现**:
- SDK 使用代码生成自动创建类型
- 支持 V1 和 V2 两套 API
- 提供服务器和客户端两种模式
- 跨语言 SDK 共享类型定义

---

## 8. 深度研究完成总结

所有 13 个 Phase 已完成！以下是最终交付物清单:

### 已完成报告

| Phase | 主题 | 文件 |
|-------|------|------|
| 1 | 基础架构 | `phase-01-basic-architecture.md` |
| 2 | Agent Loop | `phase-02-agent-loop.md` |
| 3 | Provider | `phase-03-provider.md` |
| 4 | Skill 系统 | `phase-04-skill.md` |
| 5 | MCP 集成 | `phase-05-mcp-integration.md` |
| 6 | ACP 协议 | `phase-06-acp-protocol.md` |
| 7 | 配置系统 | `phase-07-config-system.md` |
| 8 | 插件系统 | `phase-08-plugin-system.md` |
| 9 | 认证权限 | `phase-09-auth-permission-env.md` |
| 10 | 项目管理 | `phase-10-project-session-management.md` |
| 11 | 基础设施 | `phase-11-infrastructure.md` |
| 12 | Git/IDE | `phase-12-git-ide-integration.md` |
| 13 | SDK/扩展 | `phase-13-sdk-extensions.md` |

### 关键架构理解

1. **核心循环**: Session → Prompt → LLM → Tool → Session
2. **服务架构**: Effect Context + Layer 依赖注入
3. **数据存储**: SQLite + Drizzle ORM
4. **事件驱动**: PubSub + EventEmitter
5. **扩展机制**: Skill + MCP + Plugin