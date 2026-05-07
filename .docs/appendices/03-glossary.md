# OpenCode 研究术语表

> 状态: ✅ 已完成
> 最后更新: 2026-05-07

---

## A

### ACP (Agent Client Protocol)
OpenCode 作为 Agent 服务端的协议实现，允许外部客户端通过标准协议与 OpenCode 交互。
- 参考: [phase-06-acp-protocol.md](reports/phase-06-acp-protocol.md)

### Agent
在 OpenCode 中，Agent 是定义特定行为的配置单元，包含 prompt、model、permission 等。

### Agent Loop
Agent 的核心运行循环：接收用户输入 → 构建 Prompt → 调用 LLM → 执行工具 → 返回结果。

---

## B

### Bootstrap
项目初始化流程，负责加载和初始化项目所需的所有服务。
- 参考: [phase-10-project-session-management.md](reports/phase-10-project-session-management.md)

### Bus (事件总线)
OpenCode 的核心通信机制，支持实例内和全局事件发布/订阅。
- 参考: [phase-11-infrastructure.md](reports/phase-11-infrastructure.md)

---

## C

### Compaction (压缩)
当对话长度接近模型上下文限制时自动触发的上下文压缩机制，生成摘要保留关键信息。
- 参考: [conversation-storage-memory.md](reports/conversation-storage-memory.md)

---

## D

### Drizzle
OpenCode 使用的 SQLite ORM 框架。
- 参考: [phase-10-project-session-management.md](reports/phase-10-project-session-management.md)

---

## E

### Effect
OpenCode 使用的函数式编程框架，提供 Service、Layer 等机制进行依赖注入。
- 参考: [phase-01-basic-architecture.md](reports/phase-01-basic-architecture.md)

---

## G

### Git Service
Git 命令封装，提供 branch、status、diff、patch 等操作。
- 参考: [phase-12-git-ide-integration.md](reports/phase-12-git-ide-integration.md)

---

## I

### Instance / InstanceState
每个项目目录对应的独立运行实例，包含独立的服务状态。

### InstanceContext
项目实例的上下文，包含 directory、worktree、project 信息。

---

## L

### Layer
Effect 框架中的依赖组合机制，用于组合多个服务。

### LSP (Language Server Protocol)
语言服务器协议，用于代码补全、诊断等功能。
- 参考: [phase-12-git-ide-integration.md](reports/phase-12-git-ide-integration.md)

---

## M

### MCP (Model Context Protocol)
模型上下文协议，OpenCode 作为 MCP 客户端连接外部工具服务。
- 参考: [phase-05-mcp-integration.md](reports/phase-05-mcp-integration.md)

### MessageV2
OpenCode 的 V2 消息格式，包含 TextPart、ToolPart、ReasoningPart 等。

---

## P

### Permission (权限系统)
控制 Agent 可使用的工具和能力，支持 allow/deny/ask 三种动作。
- 参考: [phase-09-auth-permission-env.md](reports/phase-09-auth-permission-env.md)

### Plugin (插件系统)
OpenCode 的扩展机制，支持自定义 hooks 和服务扩展。
- 参考: [phase-08-plugin-system.md](reports/phase-08-plugin-system.md)

### Provider (LLM 提供商)
OpenCode 集成的 LLM 服务，支持 OpenAI、Google、Anthropic 等多种模型。
- 参考: [phase-03-provider.md](reports/phase-03-provider.md)

### Prompt System
OpenCode 的 Prompt 分层组装系统，根据模型 ID 选择不同的 Prompt 模板。
- 参考: [phase-prompt-system.md](reports/phase-prompt-system.md)

---

## S

### Schema
Effect 框架中的数据校验和类型定义机制。

### Session (会话)
用户与 OpenCode 的交互会话，包含消息历史、状态等。

### Skill
OpenCode 中的任务专用能力模块，定义可重用的技能。
- 参考: [phase-04-skill.md](reports/phase-04-skill.md)

### SQLite (WAL 模式)
OpenCode 使用的数据库存储模式。

---

## T

### Tool / Tool Registry
OpenCode 的工具系统，定义 read、edit、grep、bash 等内置工具。

### Turborepo
OpenCode 使用的 Monorepo 构建工具。

---

## V

### V2 消息系统
OpenCode 的新版消息格式，使用 SessionMessage 表存储。

---

## W

### Worktree
Git worktree 的封装，提供隔离的开发环境。
- 参考: [phase-10-project-session-management.md](reports/phase-10-project-session-management.md)

---

## 快速索引

| 术语 | 中文 | Phase |
|------|------|-------|
| ACP | Agent 客户端协议 | 6 |
| Agent | 智能体 | 2 |
| Bus | 事件总线 | 11 |
| Compaction | 上下文压缩 | 10 |
| Effect | 函数式框架 | 1 |
| LSP | 语言服务器协议 | 12 |
| MCP | 模型上下文协议 | 5 |
| Permission | 权限系统 | 9 |
| Plugin | 插件系统 | 8 |
| Provider | LLM 提供商 | 3 |
| Session | 会话 | 2/10 |
| Skill | 技能 | 4 |
| Worktree | 工作树 | 10 |