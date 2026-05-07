# OpenCode 研究常见问题

> 状态: ✅ 已完成
> 最后更新: 2026-05-07

---

## 1. 基础架构

### Q: OpenCode 使用什么技术栈？
A: OpenCode 使用以下核心技术：
- **运行时**: Bun
- **构建工具**: Turborepo (Monorepo)
- **函数式框架**: Effect
- **数据库**: SQLite (Drizzle ORM)
- **语言**: TypeScript

### Q: 项目的主要包有哪些？
A: 主要包包括：
- `packages/opencode` - 核心 CLI
- `packages/core` - 核心工具库
- `packages/app` - 应用层
- `packages/ui` - UI 组件
- `packages/sdk/js` - JavaScript SDK
- `packages/sdk/python` - Python SDK

---

## 2. Agent Loop

### Q: Session 主循环是如何工作的？
A: Session 主循环：
1. 接收用户输入
2. 构建 Prompt (系统提示 + 上下文 + 工具定义)
3. 调用 LLM 获取响应
4. 如果是工具调用，执行工具并返回结果
5. 重复直到完成

### Q: Prompt 是如何根据不同模型选择的？
A: 使用 `src/session/system.ts` 中的 `provider()` 函数，根据模型 ID 模式匹配：
- `claude*` → anthropic.txt
- `gpt-4/o1/o3` → beast.txt
- `gpt` (非 codex) → gpt.txt
- `gemini-*` → gemini.txt
- 其他 → default.txt

---

## 3. LLM Provider

### Q: 支持哪些 LLM 提供商？
A: 支持多种 Provider，包括：
- OpenAI (GPT 系列)
- Anthropic (Claude 系列)
- Google (Gemini 系列)
- 以及其他支持 OpenAI 兼容 API 的模型

### Q: 如何配置 API Key？
A: 通过以下方式：
- 环境变量 (如 `OPENAI_API_KEY`)
- 配置文件 `opencode.json` 中的 `providers` 字段
- 认证服务 `src/auth/index.ts` 存储

---

## 4. 扩展机制

### Q: Skill 和 Plugin 有什么区别？
A:
- **Skill**: 任务专用能力模块，是可重用的 Prompt 模板
- **Plugin**: 代码级扩展，可以添加 hooks 和服务

### Q: MCP 和 ACP 有什么区别？
A:
- **MCP (Model Context Protocol)**: OpenCode 作为客户端，连接外部工具服务
- **ACP (Agent Client Protocol)**: OpenCode 作为服务端，允许外部客户端连接

---

## 5. 权限和安全

### Q: 权限系统是如何工作的？
A: 权限系统使用规则匹配：
- `allow`: 允许执行
- `deny`: 拒绝执行
- `ask`: 询问用户

配置在 `opencode.json` 或 Agent 配置中。

### Q: 如何保护敏感数据？
A:
- API Key 存储使用 `0o600` 文件权限
- 敏感信息通过 Env 服务管理
- 支持环境变量覆盖

---

## 6. 数据存储

### Q: 会话数据存储在哪里？
A:
- 数据库: `~/.opencode/data/opencode.db` (SQLite WAL 模式)
- 表: session, message, part, todo, session_message

### Q: 什么是 Compaction？
A: Compaction 是上下文压缩机制：
- 当 token 数接近模型限制时触发
- 保留最近 N 轮对话 (tail_turns)
- 生成摘要保留关键信息
- 清理旧工具输出释放空间

---

## 7. 开发相关

### Q: 如何运行测试？
A:
```bash
cd packages/opencode
bun test
```

### Q: 如何进行类型检查？
A:
```bash
cd packages/opencode
bun typecheck
```

---

## 8. 相关资源

- 项目文档: [README.md](README.md)
- 研究计划: [01-deep-research-plan.md](01-deep-research-plan.md)
- 术语表: [03-glossary.md](03-glossary.md)