# Phase 2: Agent Loop - 会话循环 - 研究报告

> 研究时间: ~6 小时
> 状态: ✅ 已完成

## 1. Session 主循环流程

### 1.1 核心组件

Agent Loop 的核心组件位于 `src/session/` 目录：

| 组件 | 文件 | 说明 |
|------|------|------|
| Session Service | `session.ts` | 会话数据模型和 CRUD 操作 |
| Session Processor | `processor.ts` | LLM 流处理和工具调用编排 |
| Session Prompt | `prompt.ts` | Prompt 构建 (1908 行) |
| Session Run State | `run-state.ts` | 运行状态管理 |
| Message V2 | `message-v2.ts` | 消息数据模型 (1247 行) |

### 1.2 主循环流程

```
用户输入 (run command)
    ↓
SessionPrompt.loop()
    ↓
构建 Prompt (SystemPrompt + 上下文)
    ↓
LLM 调用 (LLM.Service)
    ↓
SessionProcessor 处理流
    ├── 文本响应 → 实时输出
    ├── Tool Calls → 执行工具
    └── Reasoning → 实时输出
    ↓
工具执行结果 → 返回 LLM
    ↓
循环直到完成
```

### 1.3 Session Processor 详解

**文件**: `src/session/processor.ts` (764 行)

```typescript
// 核心接口
export interface Interface {
  readonly create: (input: Input) => Effect.Effect<Handle>
}

export interface Handle {
  readonly message: MessageV2.Assistant
  readonly updateToolCall: (toolCallID, update) => Effect.Effect<MessageV2.ToolPart | undefined>
  readonly completeToolCall: (toolCallID, output) => Effect.Effect<void>
  readonly process: (streamInput: LLM.StreamInput) => Effect.Effect<Result>
}

// 处理流程
const create = Effect.fn("SessionProcessor.create")(function* (input) {
  // 1. 预捕获快照 (处理 AI SDK 内部工具执行)
  const initialSnapshot = yield* snapshot.track()
  
  // 2. 创建处理上下文
  const ctx: ProcessorContext = {
    assistantMessage: input.assistantMessage,
    sessionID: input.sessionID,
    model: input.model,
    toolcalls: {},          // 待处理工具调用
    shouldBreak: false,     // 是否终止
    snapshot: initialSnapshot,
    blocked: false,         // 是否阻塞等待
    needsCompaction: false, // 是否需要压缩
  }
  
  // 3. 处理 LLM 流事件
  const process = (streamInput) => {
    // 处理文本增量
    // 处理工具调用
    // 处理 reasoning
    // 处理错误
  }
})
```

### 1.4 关键事件

**文件**: `src/v2/session-event.ts`

```typescript
// 重要事件
SessionEvent.Step.Started     // 步骤开始
SessionEvent.Step.Ended       // 步骤结束
SessionEvent.Tool.Called      // 工具调用
SessionEvent.Tool.Input.Started
SessionEvent.Tool.Input.Ended
SessionEvent.Tool.Success
SessionEvent.Tool.Failed
SessionEvent.Reasoning.Started
SessionEvent.Reasoning.Ended
SessionEvent.Text.Started
SessionEvent.Text.Ended
SessionEvent.Compaction.Started
SessionEvent.Compaction.Ended
```

### 1.5 运行状态管理

**文件**: `src/session/run-state.ts`

```typescript
// 状态类型
type RunState = 
  | { state: "idle" }
  | { state: "running" }
  | { state: "waiting" }    // 等待工具执行
  | { state: "error"; error: Error }
```

---

## 2. Prompt 拼接系统

### 2.1 核心服务

**文件**: `src/session/prompt.ts` (1908 行)

```typescript
export interface Interface {
  readonly cancel: (sessionID: SessionID) => Effect.Effect<void>
  readonly prompt: (input: PromptInput) => Effect.Effect<MessageV2.WithParts>
  readonly loop: (input: LoopInput) => Effect.Effect<MessageV2.WithParts>
  readonly shell: (input: ShellInput) => Effect.Effect<MessageV2.WithParts>
  readonly command: (input: CommandInput) => Effect.Effect<MessageV2.WithParts>
  readonly resolvePromptParts: (template: string) => Effect.Effect<PromptInput["parts"]>
}
```

### 2.2 Prompt 构建流程

```
SessionPrompt.loop(input)
    ↓
1. 获取 Agent 信息
   - Agent.Service.get(agentID)
    ↓
2. 获取模型
   - Provider.Service.getModel()
   - 或使用默认模型
    ↓
3. 构建系统 Prompt
   - SystemPrompt.environment()  // 环境信息
   - SystemPrompt.tools()        // 可用工具
   - SystemPrompt.capabilities() // 模型能力
   - SystemPrompt.project()      // 项目信息
   - SystemPrompt.mcp()          // MCP 服务器信息
    ↓
4. 获取对话历史
   - Session.Service.messages()
   - 消息分页/截断处理
    ↓
5. 构建消息数组
   - System messages (工具定义、规则)
   - User messages (用户输入 + 历史)
   - Assistant messages (AI 回复)
    ↓
6. 返回 AI SDK 格式消息
```

### 2.3 系统提示词构建

**文件**: `src/session/system.ts`

```typescript
export interface Interface {
  readonly prompt: (model: Provider.Model, input: PromptInput) => Effect.Effect<MessageV2.Message[]>
  readonly environment: (model: Provider.Model) => Effect.Effect<string>
  readonly capabilities: (model: Provider.Model) => Effect.Effect<string>
  readonly project: () => Effect.Effect<string>
  readonly tools: (model: Provider.Model) => Effect.Effect<string>
  readonly mcp: () => Effect.Effect<string>
}
```

**Prompt 模板文件**:
- `src/session/prompt/plan.txt` - 计划模式提示
- `src/session/prompt/build-switch.txt` - 构建切换提示
- `src/session/prompt/max-steps.txt` - 最大步骤限制

### 2.4 上下文内容

Prompt 中包含的上下文：

1. **系统角色**: AI 助手的行为规范
2. **可用工具**: 所有内置 + MCP + Skill 工具
3. **MCP 工具**: 动态加载的 MCP 服务器工具
4. **项目信息**: 当前项目结构、配置
5. **模型能力**: 支持的功能 (reasoning, toolcall, etc.)
6. **对话历史**: 历史消息摘要
7. **用户输入**: 当前用户消息

### 2.5 特殊处理

- **Structured Output**: 当需要结构化输出时，添加 `StructuredOutput` 工具
- **Token 限制**: 自动处理上下文溢出
- **文件引用**: 解析 Markdown 中的文件引用

---

## 3. 工具执行机制

### 3.1 工具注册

**文件**: `src/tool/registry.ts` (356 行)

```typescript
export interface Interface {
  readonly ids: () => Effect.Effect<string[]>
  readonly all: () => Effect.Effect<Tool.Def[]>
  readonly named: () => Effect.Effect<{ task: TaskDef; read: ReadDef }>
  readonly tools: (model: { providerID, modelID, agent }) => Effect.Effect<Tool.Def[]>
}
```

### 3.2 工具类型

**内置工具** (`src/tool/`):

| 工具 | 文件 | 功能 |
|------|------|------|
| ReadTool | `read.ts` | 读取文件 |
| EditTool | `edit.ts` | 编辑文件 |
| WriteTool | `write.ts` | 写入文件 |
| GlobTool | `glob.ts` | 文件搜索 |
| GrepTool | `grep.ts` | 代码搜索 |
| ShellTool | `shell.ts` | 执行命令 |
| TaskTool | `task.ts` | 子任务 |
| QuestionTool | `question.ts` | 提问用户 |
| TodoWriteTool | `todo.ts` | 写 TODO |
| WebFetchTool | `webfetch.ts` | HTTP 请求 |
| WebSearchTool | `websearch.ts` | 网络搜索 |
| LspTool | `lsp.ts` | LSP 补全 |
| ApplyPatchTool | `apply_patch.ts` | 应用补丁 |
| SkillTool | `skill.ts` | Skill 工具 |

### 3.3 工具执行流程

```
LLM 返回 Tool Calls
    ↓
SessionProcessor.process()
    ↓
1. 解析工具名称和参数
    ↓
2. 检查工具权限
    - Permission.evaluate()
    ↓
3. 执行工具
    - Tool.execute(input, metadata)
    ↓
4. 处理执行结果
    - 成功: 返回结果给 LLM
    - 失败: 返回错误给 LLM
    ↓
5. 循环 (继续 LLM 调用)
```

### 3.4 工具权限管理

**文件**: `src/permission/`

```typescript
// 权限评估
Permission.evaluate("tool", toolName, agent.permission)
// 返回: "allow" | "deny" | "ask"

// 权限规则
type Ruleset = {
  allow?: string[]
  deny?: string[]
  ask?: string[]
}
```

### 3.5 工具定义格式

```typescript
// Tool.Def 定义
interface Tool.Def {
  name: string
  description: string
  parameters: JSONSchema7
  execute: (input, metadata) => Promise<Result>
}
```

---

## 4. 消息处理和历史

### 4.1 消息数据模型

**文件**: `src/session/message-v2.ts` (1247 行)

```typescript
// 消息类型
export type Message = 
  | UserMessage      // 用户消息
  | AssistantMessage // AI 回复
  | SystemMessage    // 系统消息

// 消息部分 (Parts)
export type Part = 
  | TextPart         // 文本
  | ToolCallPart     // 工具调用
  | ToolResultPart   // 工具结果
  | ReasoningPart    // 推理
  | FilePart         // 文件附件
  | ImagePart        // 图片
```

### 4.2 消息存储

**文件**: `src/session/session.sql.ts`

```typescript
// Drizzle Schema
const MessageTable = sqliteTable("message", {
  id: text("id").primaryKey(),
  session_id: text("session_id").notNull(),
  role: text("role").notNull(),  // user | assistant | system
  model: text("model"),           // 模型信息
  provider_id: text("provider_id"),
  // ...
})

const PartTable = sqliteTable("part", {
  id: text("id").primaryKey(),
  session_id: text("session_id").notNull(),
  message_id: text("message_id").notNull(),
  type: text("type").notNull(),  // text | tool-call | tool-result | reasoning
  // ...
})
```

### 4.3 消息分页和截断

**文件**: `src/session/compaction.ts`

```typescript
// 会话压缩触发条件
const isOverflow = (messages, maxTokens) => {
  // 检查消息总 token 数是否超过限制
}

// 压缩策略
- 删除最早的对话
- 保留系统提示
- 保留关键工具调用结果
```

### 4.4 消息更新机制

**文件**: `src/v2/session-message-updater.ts`

```typescript
// 实时更新消息部分
- 文本增量更新
- 工具调用状态更新
- 推理过程更新
- 错误信息更新
```

### 4.5 消息事件

```typescript
// 消息事件
MessageV2.Event.Created     // 消息创建
MessageV2.Event.Updated      // 消息更新
MessageV2.Event.Removed      // 消息删除
MessageV2.Event.PartUpdated  // 部分更新
MessageV2.Event.PartRemoved  // 部分删除
```

---

## 5. 关键文件索引

| 分类 | 文件 | 行数 | 说明 |
|------|------|------|------|
| **Session** | `src/session/session.ts` | 936 | 会话数据模型 |
| **Processor** | `src/session/processor.ts` | 764 | LLM 流处理 |
| **Prompt** | `src/session/prompt.ts` | 1908 | Prompt 构建 |
| **System** | `src/session/system.ts` | - | 系统提示词 |
| **Message** | `src/session/message-v2.ts` | 1247 | 消息数据模型 |
| **RunState** | `src/session/run-state.ts` | - | 运行状态 |
| **Tool Registry** | `src/tool/registry.ts` | 356 | 工具注册 |
| **Session Event** | `src/v2/session-event.ts` | - | 事件定义 |

---

## 6. 待深入研究的问题

1. **会话中断恢复**: Session 如何在中断后恢复运行?
2. **模型切换**: 运行时如何切换模型?
3. **流式输出**: 文本增量如何实时处理?
4. **错误重试**: 工具执行失败如何重试?
5. **Compaction 优化**: 会话压缩的具体策略?

---

## 7. 总结

Phase 2 完成了对 Agent Loop 的深入理解:

- ✅ 理解了 Session 主循环流程 (Prompt → LLM → Processor → Tool → LLM)
- ✅ 理解了 Prompt 拼接系统 (系统提示词、上下文构建)
- ✅ 理解了工具执行机制 (注册、权限、执行)
- ✅ 理解了消息处理和历史 (存储、分页、事件)

**下一步**: Phase 3 - Provider - LLM 提供商