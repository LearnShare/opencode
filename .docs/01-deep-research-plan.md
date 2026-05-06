# OpenCode 项目深入研究计划

## 1. 研究目标

深入理解 OpenCode 项目的技术架构、核心模块和关键功能，为后续开发维护提供知识基础。

### 1.1 核心关注领域

| 领域 | 优先级 | 说明 |
|------|--------|------|
| Agent Loop | 🔴 极高 | 会话循环、Prompt 拼接、工具调用 |
| Provider 调用 | 🔴 极高 | LLM 提供商集成、模型管理 |
| Skill 系统 | 🟠 高 | 自定义技能定义和执行 |
| MCP 集成 | 🟠 高 | Model Context Protocol 客户端 |
| ACP 协议 | 🟠 高 | Agent Client Protocol 服务端 |
| 配置系统 | 🟡 中 | 配置文件和环境变量 |

---

## 2. 研究任务总览

```
Phase 1: 基础架构 ✅ (已完成)
    ├── 1.1 项目结构理解 ✅
    ├── 1.2 入口点和启动流程 ✅
    └── 1.3 Effect 框架基础 ✅

Phase 2: Agent Loop - 会话循环 ✅ (已完成)
    ├── 2.1 Session 主循环流程 ✅
    ├── 2.2 Prompt 拼接系统 ✅
    ├── 2.3 工具执行机制 ✅
    └── 2.4 消息处理和历史 ✅

Phase 3: Provider - LLM 提供商 ✅ (已完成)
    ├── 3.1 Provider 接口和架构 ✅
    ├── 3.2 支持的模型列表 ✅
    ├── 3.3 认证机制 ✅
    └── 3.4 模型选择逻辑 ✅

Phase 4: Skill 系统 ✅ (已完成)
    ├── 4.1 Skill 核心概念 ✅
    └── 4.2 Skill 与工具集成 ✅

Phase 5: MCP 集成 (预计 6-8 小时)
    ├── 5.1 MCP 核心实现
    └── 5.2 MCP 配置和管理

Phase 6: ACP 协议 (预计 4-6 小时)
    ├── 6.1 ACP 协议规范
    └── 6.2 ACP 服务端实现

Phase 7: 配置系统 (预计 3-4 小时)

Phase 8: 插件系统 (预计 2-3 小时)

Phase 9: 认证和权限 (预计 3-4 小时)
    ├── 9.1 认证系统 (Auth)
    ├── 9.2 权限系统 (Permission)
    └── 9.3 环境变量 (Env)

Phase 10: 项目和会话管理 (预计 4-5 小时)
    ├── 10.1 项目引导 (Bootstrap)
    ├── 10.2 工作目录 (Worktree)
    └── 10.3 会话存储 (Storage/Session)

Phase 11: 基础设施服务 (预计 3-4 小时)
    ├── 11.1 事件总线 (Bus)
    ├── 11.2 日志系统
    └── 11.3 错误处理

Phase 12: Git 和 IDE 集成 (预计 2-3 小时)
    ├── 12.1 Git 集成
    └── 12.2 LSP 集成

Phase 13: SDK 和扩展 (预计 2-3 小时)
    ├── 13.1 JavaScript SDK
    └── 13.2 Python SDK
```

**已完成**: Phase 1 (3-5 小时)
**预计总时长**: 60-75 小时

---

## 3. 详细任务说明

### Phase 1: 基础架构 ✅

#### 任务 1.1: 项目结构理解 [优先级: 🔴 高] ✅

**目标**: 理解项目的整体目录结构和包之间的关系

**具体任务**:
- [x] 阅读根目录 `package.json` 了解 workspace 配置
- [x] 查看 `turbo.json` 了解构建配置
- [x] 探索 `packages/` 目录结构，列出所有包
- [x] 理解每个主要包的职责 (opencode, core, app, ui 等)
- [x] 了解包之间的依赖关系

**关键文件**:
- `package.json` - workspace 定义
- `turbo.json` - 构建配置
- `packages/opencode/package.json` - 核心 CLI
- `packages/core/package.json` - 核心工具库

**交付物**: 项目结构文档 (包括包关系图) ✅

**预计时长**: 1-2 小时

---

#### 任务 1.2: 入口点和启动流程 [优先级: 🔴 高] ✅

**目标**: 理解应用如何启动，CLI 和 Server 如何工作

**具体任务**:
- [x] 找到 CLI 入口点 `src/index.ts`，理解 main 函数
- [x] 理解命令解析流程 - 查看 `cli/` 目录
- [x] 理解 Server 启动流程 - 查看 `server/` 目录
- [x] 理解 TUI (Terminal UI) 启动流程
- [x] 理解不同模式 (cli, server, tui, acp) 的切换逻辑

**关键文件**:
- `packages/opencode/src/index.ts`
- `packages/opencode/src/cli/cmd/` - 各命令实现
- `packages/opencode/src/server/` - 服务端

**交付物**: 启动流程图和模式切换说明 ✅

**预计时长**: 1-2 小时

---

#### 任务 1.3: Effect 框架基础 [优先级: 🟡 中] ✅

**目标**: 理解项目使用的函数式编程框架

**具体任务**:
- [x] 了解 Effect Context 和 Service 的使用方式
- [x] 理解 Layer 机制 - 如何组合服务
- [x] 理解 Effect.gen 用法 - 异步编程模式
- [x] 理解 Effect.fn 和 Effect.fnUntraced 的区别
- [x] 查看 `AGENTS.md` 中的 Effect 规范

**关键文件**:
- `packages/opencode/AGENTS.md` - Effect 规范
- `packages/opencode/src/effect/` - Effect 相关

**交付物**: Effect 使用简明指南 ✅

**预计时长**: 1 小时

---

### Phase 2: Agent Loop - 会话循环

#### 任务 2.1: Session 主循环流程 [优先级: 🔴 极高]

**目标**: 理解 Agent 如何与用户交互并处理任务

**具体任务**:
- [ ] 阅读 `session/session.ts` 完整内容 (分多次阅读)
- [ ] 理解会话状态管理 - `run-state.ts`
- [ ] 理解用户输入处理流程
- [ ] 理解响应的生成和发送机制
- [ ] 理解 V2 会话与 V1 的区别 - 查看 `v2/session.ts`

**关键文件**:
- `packages/opencode/src/session/session.ts` (~1500 行)
- `packages/opencode/src/session/run-state.ts`
- `packages/opencode/src/v2/session.ts`

**交付物**: Session 循环流程图和核心代码解读

**预计时长**: 3-4 小时

---

#### 任务 2.2: Prompt 拼接系统 [优先级: 🔴 极高]

**目标**: 理解如何构建发送给 LLM 的提示词

**具体任务**:
- [ ] 阅读 `session/prompt.ts` 完整内容
- [ ] 理解系统提示词构建 - `system.ts`
- [ ] 理解上下文包含内容 (文件、工具、消息等)
- [ ] 理解对话历史处理
- [ ] 查看 V2 版本的 prompt 构建 - `v2/session-prompt.ts`

**关键文件**:
- `packages/opencode/src/session/prompt.ts` (~1800 行)
- `packages/opencode/src/session/system.ts`
- `packages/opencode/src/v2/session-prompt.ts`

**交付物**: Prompt 构建流程图和组件说明

**预计时长**: 2-3 小时

---

#### 任务 2.3: 工具执行机制 [优先级: 🔴 高]

**目标**: 理解 Agent 如何调用工具

**具体任务**:
- [ ] 理解工具注册机制 - `tool/registry.ts`
- [ ] 理解工具调用流程 - `tool/tool.ts`
- [ ] 理解工具结果处理
- [ ] 理解工具权限管理 - `permission/` 目录
- [ ] 了解内置工具实现 (read, edit, glob, grep, bash 等)
- [ ] 理解工具超时和错误处理

**关键文件**:
- `packages/opencode/src/tool/registry.ts`
- `packages/opencode/src/tool/tool.ts`
- `packages/opencode/src/permission/`
- `packages/opencode/src/tool/*.ts` - 各工具实现

**交付物**: 工具系统架构文档

**预计时长**: 2-3 小时

---

#### 任务 2.4: 消息处理和历史 [优先级: 🟡 中]

**目标**: 理解消息如何存储和管理

**具体任务**:
- [ ] 理解消息数据结构 - `session/message.ts`
- [ ] 理解消息持久化 - `storage/` 目录
- [ ] 理解消息分页和截断机制
- [ ] 理解会话压缩 - `session/compaction.ts`
- [ ] 理解消息更新机制 - `session-message-updater.ts`

**关键文件**:
- `packages/opencode/src/session/message.ts`
- `packages/opencode/src/storage/`
- `packages/opencode/src/session/compaction.ts`
- `packages/opencode/src/v2/session-message.ts`

**交付物**: 消息管理说明文档

**预计时长**: 1-2 小时

---

### Phase 3: Provider - LLM 提供商

#### 任务 3.1: Provider 接口和架构 [优先级: 🔴 极高]

**目标**: 理解 Provider 系统的核心设计

**具体任务**:
- [ ] 阅读 `provider/provider.ts` 完整内容 (分多次阅读)
- [ ] 理解 Provider 接口定义
- [ ] 理解模型加载机制
- [ ] 理解 Provider 优先级 (env > config > custom > api)
- [ ] 理解自定义模型加载 - `custom()` 函数

**关键文件**:
- `packages/opencode/src/provider/provider.ts` (~1500+ 行)
- `packages/opencode/src/provider/schema.ts`

**交付物**: Provider 架构文档

**预计时长**: 2-3 小时

---

#### 任务 3.2: 支持的模型列表 [优先级: 🔴 高]

**目标**: 了解支持哪些模型

**具体任务**:
- [ ] 查看 `provider/models.ts` 了解模型定义格式
- [ ] 理解模型配置结构
- [ ] 了解各模型的 capabilities (reasoning, toolcall 等)
- [ ] 理解模型的 variants 机制

**关键文件**:
- `packages/opencode/src/provider/models.ts`
- `packages/opencode/src/provider/transform.ts`

**交付物**: 模型列表和配置说明

**预计时长**: 1-2 小时

---

#### 任务 3.3: Provider 认证 [优先级: 🟠 中]

**目标**: 理解如何配置认证

**具体任务**:
- [ ] 理解 API Key 配置方式
- [ ] 理解 OAuth 流程
- [ ] 理解环境变量支持
- [ ] 理解各 Provider 的特殊认证方式

**关键文件**:
- `packages/opencode/src/provider/auth.ts`
- `packages/opencode/src/auth/`
- `packages/opencode/src/env/`

**交付物**: 认证配置指南

**预计时长**: 1-2 小时

---

#### 任务 3.4: 模型选择逻辑 [优先级: 🟠 中]

**目标**: 理解如何选择合适的模型

**具体任务**:
- [ ] 理解默认模型选择
- [ ] 理解上下文窗口限制处理
- [ ] 理解成本优化选择
- [ ] 理解模型切换机制

**关键文件**:
- `packages/opencode/src/session/prompt.ts` 中模型选择部分
- `packages/opencode/src/provider/provider.ts` 中模型查询部分

**交付物**: 模型选择流程说明

**预计时长**: 1 小时

---

### Phase 4: Skill 系统

#### 任务 4.1: Skill 核心概念 [优先级: 🔴 高]

**目标**: 理解 Skill 与工具的区别

**具体任务**:
- [ ] 阅读 `skill/index.ts` 完整内容
- [ ] 理解 Skill 定义 (SKILL.md 格式)
- [ ] 理解 Skill 发现机制 - `skill/discovery.ts`
- [ ] 理解 Skill 加载流程
- [ ] 理解 Skill 执行模型

**关键文件**:
- `packages/opencode/src/skill/index.ts` (~300 行)
- `packages/opencode/src/skill/discovery.ts`

**交付物**: Skill 系统架构文档

**预计时长**: 2-3 小时

---

#### 任务 4.2: Skill 与工具集成 [优先级: 🟠 中]

**目标**: 理解 Skill 如何作为工具暴露

**具体任务**:
- [ ] 理解 Skill 注册为工具 - `tool/skill.ts`
- [ ] 理解 Skill 参数处理
- [ ] 理解 Skill 权限评估

**关键文件**:
- `packages/opencode/src/tool/skill.ts`

**交付物**: Skill 集成说明

**预计时长**: 1-2 小时

---

### Phase 5: MCP 集成

#### 任务 5.1: MCP 核心实现 [优先级: 🔴 高]

**目标**: 理解 MCP 协议实现

**具体任务**:
- [ ] 阅读 `mcp/index.ts` 完整内容 (~930 行)
- [ ] 理解 MCP 客户端初始化
- [ ] 理解远程和本地连接方式
- [ ] 理解 MCP 工具调用流程
- [ ] 理解 MCP 资源管理
- [ ] 理解工具列表变更通知处理

**关键文件**:
- `packages/opencode/src/mcp/index.ts`
- `packages/opencode/src/mcp/auth.ts`

**交付物**: MCP 实现文档

**预计时长**: 3-4 小时

---

#### 任务 5.2: MCP 配置和管理 [优先级: 🟠 中]

**目标**: 理解 MCP 服务器配置

**具体任务**:
- [ ] 理解 MCP 配置格式 - `config/mcp.ts`
- [ ] 理解 MCP 状态管理
- [ ] 理解 MCP OAuth 流程
- [ ] 理解 MCP API 路由

**关键文件**:
- `packages/opencode/src/config/mcp.ts`
- `packages/opencode/src/server/routes/instance/mcp.ts`
- `packages/opencode/src/mcp/oauth-provider.ts`

**交付物**: MCP 配置指南

**预计时长**: 2-3 小时

---

### Phase 6: ACP 协议

#### 任务 6.1: ACP 协议规范 [优先级: 🔴 高]

**目标**: 理解 ACP 协议

**具体任务**:
- [ ] 了解 ACP 协议用途
- [ ] 理解协议消息类型
- [ ] 理解会话管理方式

**关键文件**:
- `@agentclientprotocol/sdk` 包 (外部依赖)

**交付物**: ACP 协议简述

**预计时长**: 1-2 小时

---

#### 任务 6.2: ACP 服务端实现 [优先级: 🔴 高]

**目标**: 理解 ACP 服务端如何工作

**具体任务**:
- [ ] 阅读 `acp/agent.ts` 完整内容 (~1800+ 行)
- [ ] 理解会话管理 - `acp/session.ts`
- [ ] 理解客户端连接处理
- [ ] 理解事件订阅和处理
- [ ] 理解权限处理

**关键文件**:
- `packages/opencode/src/acp/agent.ts`
- `packages/opencode/src/acp/session.ts`
- `packages/opencode/src/cli/cmd/acp.ts`

**交付物**: ACP 服务端架构文档

**预计时长**: 3-4 小时

---

### Phase 7: 配置系统

#### 任务 7.1: 配置文件 [优先级: 🟡 中]

**目标**: 理解配置系统

**具体任务**:
- [ ] 理解 opencode.json 结构
- [ ] 理解配置加载流程
- [ ] 理解配置验证
- [ ] 理解配置优先级

**关键文件**:
- `packages/opencode/src/config/config.ts`
- `packages/opencode/src/config/*.ts` - 各配置模块

**交付物**: 配置参考文档

**预计时长**: 2-3 小时

---

### Phase 8: 插件系统

#### 任务 8.1: 插件核心机制 [优先级: 🟡 中]

**目标**: 理解插件系统

**具体任务**:
- [ ] 理解插件接口定义 - `plugin/index.ts`
- [ ] 理解插件加载流程 - `plugin/loader.ts`
- [ ] 理解插件 API 暴露
- [ ] 理解插件的 provider 扩展
- [ ] 理解插件的 auth 扩展

**关键文件**:
- `packages/opencode/src/plugin/index.ts`
- `packages/opencode/src/plugin/loader.ts`
- `packages/opencode/src/plugin/install.ts`

**交付物**: 插件系统说明

**预计时长**: 2-3 小时

---

### Phase 9: 认证和权限

#### 任务 9.1: 认证系统 (Auth) [优先级: 🟡 中]

**目标**: 理解认证管理系统

**具体任务**:
- [ ] 理解 Auth 接口和实现
- [ ] 理解 API Key 存储
- [ ] 理解 OAuth 令牌管理
- [ ] 理解认证状态持久化

**关键文件**:
- `packages/opencode/src/auth/index.ts`
- `packages/opencode/src/auth/oauth.ts`

**交付物**: 认证系统说明

**预计时长**: 1-2 小时

---

#### 任务 9.2: 权限系统 (Permission) [优先级: 🟡 中]

**目标**: 理解工具和技能权限控制

**具体任务**:
- [ ] 理解权限评估机制
- [ ] 理解权限规则定义
- [ ] 理解权限自动响应
- [ ] 理解权限持久化

**关键文件**:
- `packages/opencode/src/permission/index.ts`
- `packages/opencode/src/permission/ask.ts`

**交付物**: 权限系统说明

**预计时长**: 1-2 小时

---

#### 任务 9.3: 环境变量 (Env) [优先级: 🟡 低]

**目标**: 理解环境变量管理

**具体任务**:
- [ ] 理解环境变量加载
- [ ] 理解环境变量优先级
- [ ] 理解敏感信息处理

**关键文件**:
- `packages/opencode/src/env/index.ts`

**交付物**: 环境变量说明

**预计时长**: 1 小时

---

### Phase 10: 项目和会话管理

#### 任务 10.1: 项目引导 (Bootstrap) [优先级: 🟡 中]

**目标**: 理解项目初始化流程

**具体任务**:
- [ ] 理解项目发现机制
- [ ] 理解项目初始化
- [ ] 理解依赖安装
- [ ] 理解 Git 初始化

**关键文件**:
- `packages/opencode/src/project/bootstrap.ts`
- `packages/opencode/src/project/index.ts`

**交付物**: 项目引导说明

**预计时长**: 1-2 小时

---

#### 任务 10.2: 工作目录 (Worktree) [优先级: 🟡 中]

**目标**: 理解工作目录管理

**具体任务**:
- [ ] 理解 Worktree 概念
- [ ] 理解工作目录切换
- [ ] 理解多项目支持

**关键文件**:
- `packages/opencode/src/worktree/index.ts`
- `packages/opencode/src/project/worktree.ts`

**交付物**: Worktree 说明

**预计时长**: 1-2 小时

---

#### 任务 10.3: 会话存储 (Storage/Session) [优先级: 🟡 低]

**目标**: 理解数据持久化

**具体任务**:
- [ ] 理解数据库 schema
- [ ] 理解 Drizzle 使用
- [ ] 理解数据迁移
- [ ] 理解会话数据存储结构

**关键文件**:
- `packages/opencode/src/storage/`
- `packages/opencode/src/session/session.sql.ts`

**交付物**: 存储系统说明

**预计时长**: 1-2 小时

---

### Phase 11: 基础设施服务

#### 任务 11.1: 事件总线 (Bus) [优先级: 🟡 中]

**目标**: 理解事件通信机制

**具体任务**:
- [ ] 理解 Bus 接口和实现
- [ ] 理解事件发布/订阅
- [ ] 理解事件类型定义

**关键文件**:
- `packages/opencode/src/bus/index.ts`
- `packages/opencode/src/bus/bus-event.ts`

**交付物**: 事件总线说明

**预计时长**: 1 小时

---

#### 任务 11.2: 日志系统 [优先级: 🟡 低]

**目标**: 理解日志记录

**具体任务**:
- [ ] 理解日志创建和使用
- [ ] 理解日志级别
- [ ] 理解日志格式

**关键文件**:
- `@opencode-ai/core/util/log.ts`

**交付物**: 日志说明

**预计时长**: 0.5 小时

---

#### 任务 11.3: 错误处理 [优先级: 🟡 低]

**目标**: 理解错误处理机制

**具体任务**:
- [ ] 理解错误类型定义
- [ ] 理解错误传播机制
- [ ] 理解用户友好的错误消息

**关键文件**:
- `packages/opencode/src/util/error.ts`
- `packages/opencode/src/util/named-schema-error.ts`

**交付物**: 错误处理说明

**预计时长**: 0.5 小时

---

### Phase 12: Git 和 IDE 集成

#### 任务 12.1: Git 集成 [优先级: 🟡 低]

**目标**: 理解 Git 操作

**具体任务**:
- [ ] 理解 Git 命令封装
- [ ] 理解差异计算 (diff)
- [ ] 理解补丁应用

**关键文件**:
- `packages/opencode/src/git/index.ts`
- `packages/opencode/src/git/diff.ts`
- `packages/opencode/src/tool/apply_patch.ts`

**交付物**: Git 集成说明

**预计时长**: 1 小时

---

#### 任务 12.2: LSP 集成 [优先级: 🟡 低]

**目标**: 理解语言服务器协议

**具体任务**:
- [ ] 理解 LSP 客户端实现
- [ ] 理解代码补全
- [ ] 理解代码诊断

**关键文件**:
- `packages/opencode/src/lsp/index.ts`

**交付物**: LSP 集成说明

**预计时长**: 1 小时

---

### Phase 13: SDK 和扩展

#### 任务 13.1: JavaScript SDK [优先级: 🟡 低]

**目标**: 理解 JS SDK

**具体任务**:
- [ ] 理解 SDK 结构
- [ ] 理解 API 客户端
- [ ] 理解类型定义

**关键文件**:
- `packages/sdk/js/src/`
- `packages/opencode/src/share/session.ts`

**交付物**: SDK 说明

**预计时长**: 1 小时

---

#### 任务 13.2: Python SDK [优先级: 🟡 低]

**目标**: 理解 Python SDK

**具体任务**:
- [ ] 理解 Python SDK 结构
- [ ] 理解 API 调用方式

**关键文件**:
- `packages/sdk/python/`

**交付物**: Python SDK 说明

**预计时长**: 1 小时

---

## 4. 推荐学习路径

| 周次 | 内容 | 产出 |
|------|------|------|
| 第1周 | Phase 1-2 | 项目结构 + Agent Loop |
| 第2周 | Phase 3-4 | Provider + Skill |
| 第3周 | Phase 5-6 | MCP + ACP |
| 第4周 | Phase 7-9 | 配置 + 插件 + 认证 |
| 第5周 | Phase 10-13 | 项目管理 + 基础设施 + 扩展 |

---

## 5. 研究方法建议

### 5.1 代码阅读方法

1. **先整体后局部**: 先理解文件在整体架构中的位置，再深入细节
2. **分多次阅读**: 大文件不要一次性读完，分章节逐步理解
3. **配合测试**: 阅读代码时查看相关测试用例帮助理解
4. **动手验证**: 通过修改代码或添加日志来验证理解

### 5.2 文档记录建议

1. 每个 Phase 完成后写小结
2. 记录关键代码位置和行号
3. 记录未解决的问题
4. 整理术语表

---

## 6. 文档交付要求

**报告必须保存为 Markdown 文档，分类保存到 `.docs/` 目录**：

命名规则: 使用 `abc-xyz.md` 格式 (小写字母 + 连字符)

```
.docs/
├── reports/
│   ├── phase-01-basic-architecture.md
│   ├── phase-02-agent-loop.md
│   └── ...
├── plans/
│   └── 01-deep-research-plan.md
└── assets/ (可选：图片、图表)
```

**每个 Phase 的报告需包含**：
1. 核心发现和架构理解
2. 关键代码文件和行号
3. 流程图或架构图 (如有)
4. 重要概念解释
5. 待深入研究的问题

---

## 7. 交付物清单

- [x] 项目结构文档 (Phase 1)
- [x] 启动流程图 (Phase 1)
- [x] Effect 使用指南 (Phase 1)
- [x] Session 循环流程图 (Phase 2)
- [x] Prompt 构建流程图 (Phase 2)
- [x] 工具系统架构文档 (Phase 2)
- [x] Provider 架构文档 (Phase 3)
- [x] 模型列表说明 (Phase 3)
- [x] 认证配置指南 (Phase 3)
- [x] Skill 系统架构文档 (Phase 4)
- [ ] MCP 实现文档 (Phase 5)
- [ ] ACP 服务端架构文档 (Phase 6)
- [ ] 配置参考文档 (Phase 7)
- [ ] 插件系统说明 (Phase 8)
- [ ] 认证权限说明 (Phase 9)
- [ ] 项目管理说明 (Phase 10)
- [ ] 基础设施说明 (Phase 11)
- [ ] 集成说明 (Phase 12)
- [ ] SDK 说明 (Phase 13)
- [ ] 术语表和常见问题 (全程)