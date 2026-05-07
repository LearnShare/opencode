# OpenCode 项目研究报告

> 状态: ✅ 全部完成
> 最后更新: 2026-05-07

---

## 文档概览

本目录包含 OpenCode 项目的深入研究报告，涵盖项目架构、核心模块、集成功能等各个方面。

---

## 目录结构

```
.docs/
├── readme.md                      # 本文档
├── plan/                          # 计划文档
│   ├── 01-deep-research-plan.md
│   └── 02-documentation-reorganization-plan.md
│
├── architecture/                 # 架构文档
│   └── architecture.md
│
├── core/                          # 核心模块
│   ├── agent-loop.md
│   ├── prompt-system.md
│   └── session-storage.md
│
├── integration/                   # 集成模块
│   ├── provider.md
│   ├── skill.md
│   ├── mcp.md
│   ├── acp.md
│   ├── config.md
│   └── plugin.md
│
├── security/                      # 安全模块
│   └── auth-permission.md
│
├── project/                       # 项目管理
│   └── project-session.md
│
├── infrastructure/                # 基础设施
│   └── infrastructure.md
│
├── integration-ext/                # 外部集成
│   └── git-ide.md
│
├── sdk/                           # SDK
│   └── sdk.md
│
└── appendices/                    # 附录
    ├── 03-glossary.md
    └── 04-faq.md
```

---

## 研究计划

- **[plan/01-deep-research-plan.md](plan/01-deep-research-plan.md)** - 研究计划总览，包含 13 个 Phase 的任务清单和完成状态
- **[plan/02-documentation-reorganization-plan.md](plan/02-documentation-reorganization-plan.md)** - 文档整理计划

---

## 报告目录

### architecture/ - 基础架构

| 文档 | 说明 |
|------|------|
| [architecture/architecture.md](architecture/architecture.md) | 项目结构、入口点、Effect 框架 |

### core/ - 核心模块

| 文档 | 说明 |
|------|------|
| [core/agent-loop.md](core/agent-loop.md) | Session 主循环流程 |
| [core/prompt-system.md](core/prompt-system.md) | Prompt 拼接系统 |
| [core/session-storage.md](core/session-storage.md) | 会话存储与记忆机制 |

### integration/ - 集成模块

| 文档 | 说明 |
|------|------|
| [integration/provider.md](integration/provider.md) | LLM Provider 接口 |
| [integration/skill.md](integration/skill.md) | Skill 核心概念 |
| [integration/mcp.md](integration/mcp.md) | MCP 协议实现 |
| [integration/acp.md](integration/acp.md) | ACP 协议规范 |
| [integration/config.md](integration/config.md) | 配置系统 |
| [integration/plugin.md](integration/plugin.md) | 插件系统 |

### security/ - 安全模块

| 文档 | 说明 |
|------|------|
| [security/auth-permission.md](security/auth-permission.md) | 认证、权限、环境变量 |

### project/ - 项目管理

| 文档 | 说明 |
|------|------|
| [project/project-session.md](project/project-session.md) | Bootstrap、Worktree、存储 |

### infrastructure/ - 基础设施

| 文档 | 说明 |
|------|------|
| [infrastructure/infrastructure.md](infrastructure/infrastructure.md) | 事件总线、日志、错误处理 |

### integration-ext/ - 外部集成

| 文档 | 说明 |
|------|------|
| [integration-ext/git-ide.md](integration-ext/git-ide.md) | Git、LSP 集成 |

### sdk/ - SDK

| 文档 | 说明 |
|------|------|
| [sdk/sdk.md](sdk/sdk.md) | JavaScript SDK、Python SDK |

### appendices/ - 附录

| 文档 | 说明 |
|------|------|
| [appendices/03-glossary.md](appendices/03-glossary.md) | 术语表 |
| [appendices/04-faq.md](appendices/04-faq.md) | 常见问题 |

---

## 文档规范

所有报告文档遵循以下标准格式:

```markdown
# 标题

> 研究时间: ~X 小时
> 状态: ✅ 已完成
> 关联 Phase: X

---

## 1. 概述

## 2. 核心概念

## 3. 实现细节

## 4. 关键文件索引

## 5. 总结
```

---

## 按功能模块快速索引

| 模块 | 目录 | 文档 |
|------|------|------|
| 基础架构 | architecture | architecture.md |
| Agent 循环 | core | agent-loop.md, prompt-system.md |
| LLM 集成 | integration | provider.md |
| 扩展机制 | integration | skill.md, mcp.md, acp.md, plugin.md |
| 安全认证 | security | auth-permission.md |
| 项目管理 | project | project-session.md |
| 基础设施 | infrastructure | infrastructure.md |
| 外部集成 | integration-ext | git-ide.md |
| 开发者 API | sdk | sdk.md |

---

## 附录

- [appendices/03-glossary.md](appendices/03-glossary.md) - 术语表
- [appendices/04-faq.md](appendices/04-faq.md) - 常见问题