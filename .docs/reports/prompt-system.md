# Prompt 分层与组装逻辑深入研究报告

> 研究时间: ~3 小时
> 状态: ✅ 已完成

---

## 1. Prompt 层级架构总览

OpenCode 的 Prompt 系统采用**分层组装**架构，从上到下分为以下层级:

```
┌─────────────────────────────────────────────────────────────┐
│ Layer 1: Model-Specific Prompt (模型特定提示)              │
│   - 根据模型 ID 选择: anthropic, gpt, gemini, beast, etc.  │
├─────────────────────────────────────────────────────────────┤
│ Layer 2: System Prompt (系统级提示)                       │
│   - environment() - 环境信息                               │
│   - skills() - 可用技能列表                               │
├─────────────────────────────────────────────────────────────┤
│ Layer 3: Tool Definitions (工具定义)                       │
│   - 所有内置工具 + MCP 工具 + Skill 工具                  │
├─────────────────────────────────────────────────────────────┤
│ Layer 4: Context (上下文)                                 │
│   - 项目信息、文件结构、配置                              │
├─────────────────────────────────────────────────────────────┤
│ Layer 5: Mode-Specific (模式特定)                         │
│   - plan.txt, build-switch.txt, max-steps.txt             │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. 核心组装逻辑

### 2.1 SystemPrompt 服务

**文件**: `packages/opencode/src/session/system.ts` (84 行)

```typescript
// 模型选择函数 - 根据模型 ID 选择对应的 Prompt
export function provider(model: Provider.Model) {
  // GPT-4, o1, o3 系列 -> beast 模式
  if (model.api.id.includes("gpt-4") || model.api.id.includes("o1") || model.api.id.includes("o3"))
    return [PROMPT_BEAST]
  
  // GPT 系列 -> gpt 模式
  if (model.api.id.includes("gpt")) {
    if (model.api.id.includes("codex")) return [PROMPT_CODEX]
    return [PROMPT_GPT]
  }
  
  // Gemini -> gemini 模式
  if (model.api.id.includes("gemini-")) return [PROMPT_GEMINI]
  
  // Claude -> anthropic 模式
  if (model.api.id.includes("claude")) return [PROMPT_ANTHROPIC]
  
  // Trinity -> trinity 模式
  if (model.api.id.toLowerCase().includes("trinity")) return [PROMPT_TRINITY]
  
  // Kimi -> kimi 模式
  if (model.api.id.toLowerCase().includes("kimi")) return [PROMPT_KIMI]
  
  // 默认 -> default 模式
  return [PROMPT_DEFAULT]
}

// 环境信息构建
environment: Effect.fn("SystemPrompt.environment")(function* (model) {
  return [
    `You are powered by the model named ${model.api.id}. The exact model ID is ${model.providerID}/${model.api.id}`,
    `Here is some useful information about the environment you are running in:`,
    `<env>`,
    `  Working directory: ${ctx.directory}`,
    `  Workspace root folder: ${ctx.worktree}`,
    `  Is directory a git repo: ${ctx.project.vcs === "git" ? "yes" : "no"}`,
    `  Platform: ${process.platform}`,
    `  Today's date: ${new Date().toDateString()}`,
    `</env>`,
  ].join("\n")
})

// 技能列表构建
skills: Effect.fn("SystemPrompt.skills")(function* (agent) {
  const list = yield* skill.available(agent)
  return [
    "Skills provide specialized instructions and workflows for specific tasks.",
    "Use the skill tool to load a skill when a task matches its description.",
    Skill.fmt(list, { verbose: true }),
  ].join("\n")
})
```

### 2.2 SessionPrompt 流程

**文件**: `packages/opencode/src/session/prompt.ts` (1908 行) 中的核心逻辑:

```typescript
// 主要接口
export interface Interface {
  readonly cancel: (sessionID: SessionID) => Effect.Effect<void>
  readonly prompt: (input: PromptInput) => Effect.Effect<MessageV2.WithParts>
  readonly loop: (input: LoopInput) => Effect.Effect<MessageV2.WithParts>
  readonly shell: (input: ShellInput) => Effect.Effect<MessageV2.WithParts>
  readonly command: (input: CommandInput) => Effect.Effect<MessageV2.WithParts>
  readonly resolvePromptParts: (template: string) => Effect.Effect<PromptInput["parts"]>
}

// 构建流程 (简化版)
const prompt = Effect.gen(function* () {
  // 1. 获取 Agent 信息
  const agentInfo = yield* Agent.Service.get(agentID)
  
  // 2. 获取模型
  const model = yield* Provider.Service.getModel(providerID, modelID)
  
  // 3. 获取系统 Prompt (基于模型选择)
  const systemPrompt = SystemPrompt.provider(model)
  
  // 4. 获取工具定义
  const tools = yield* ToolRegistry.Service.tools({ providerID, modelID, agent: agentInfo })
  
  // 5. 获取对话历史
  const messages = yield* Session.Service.messages({ sessionID, limit })
  
  // 6. 组装完整消息
  return {
    system: [...systemPrompt, ...envPrompt, ...skillsPrompt],
    messages: [...history, userInput],
    tools,
  }
})
```

---

## 3. 全部内置 Prompt 完整内容

### 3.1 default.txt (默认模式)

**用途**: 默认回退模式，适用于未匹配到特定模型的场景

**特点**: 
- 简洁直接的风格
- 强调最少化输出
- 4 行以内响应原则

**文件**: `packages/opencode/src/session/prompt/default.txt`

---

### 3.2 anthropic.txt (Claude 模式)

**用途**: Claude 系列模型 (claude)

**特点**:
- 强调 TodoWrite 任务管理
- 鼓励使用 Task 工具
- 包含专业客观性原则

**文件**: `packages/opencode/src/session/prompt/anthropic.txt` (105 行)

---

### 3.3 gpt.txt (GPT 模式)

**用途**: OpenAI GPT 系列 (gpt, 非 codex)

**特点**:
- 极度简洁的响应风格
- 强调最小改动原则
- commentary/final 双通道输出

**文件**: `packages/opencode/src/session/prompt/gpt.txt` (107 行)

---

### 3.4 beast.txt ( Beast 模式)

**用途**: GPT-4, o1, o3 系列

**特点**:
- 强调持续迭代直到完成
- 强制网络研究
- 详细的 10 步工作流

**文件**: `packages/opencode/src/session/prompt/beast.txt` (147 行)

---

### 3.5 gemini.txt (Gemini 模式)

**用途**: Google Gemini 系列

**特点**:
- 严格的项目规范遵循
- 强调路径构建
- 新应用开发的完整流程

**文件**: `packages/opencode/src/session/prompt/gemini.txt` (155 行)

---

### 3.6 trinity.txt (Trinity 模式)

**用途**: Trinity 模型

**特点**:
- 类似 default，但限制单工具调用
- 强调一个工具 per message

**文件**: `packages/opencode/src/session/prompt/trinity.txt` (97 行)

---

### 3.7 kimi.txt (Kimi 模式)

**用途**: Kimi 模型

**特点**:
- 中文支持
- 强调 AGENTS.md 文件
- 研究和数据处理指南

**文件**: `packages/opencode/src/session/prompt/kimi.txt` (95 行)

---

### 3.8 codex.txt (Codex 模式)

**用途**: GPT Codex 系列

**特点**:
- 前端设计特别指南
- 严格的文件引用格式
- 工作流卫生规范

**文件**: `packages/opencode/src/session/prompt/codex.txt` (79 行)

---

### 3.9 copilot-gpt-5.txt (Copilot 模式)

**用途**: GitHub Copilot

**特点**:
- 最复杂的工作流
- 包含多个子指令模块
- 详细的代码搜索指南

**文件**: `packages/opencode/src/session/prompt/copilot-gpt-5.txt` (143 行)

---

### 3.10 plan.txt (计划模式)

**用途**: 用户选择计划模式时

**特点**:
- 只读模式，禁止修改
- 强调理解、规划而非执行

**文件**: `packages/opencode/src/session/prompt/plan.txt`

---

### 3.11 plan-reminder-anthropic.txt (Anthropic 计划模式)

**用途**: Anthropic 模型的计划模式

**特点**:
- 5 阶段增强规划工作流
- 最多 3 个并行 Explore 代理
- 强制创建计划文件

**文件**: `packages/opencode/src/session/prompt/plan-reminder-anthropic.txt` (67 行)

---

### 3.12 build-switch.txt (构建切换模式)

**用途**: 从计划模式切换到构建模式

**特点**:
- 简单通知：可以开始修改

**文件**: `packages/opencode/src/session/prompt/build-switch.txt`

---

### 3.13 max-steps.txt (最大步骤限制)

**用途**: 达到最大步数限制

**特点**:
- 强制只读
- 要求总结已完成工作

**文件**: `packages/opencode/src/session/prompt/max-steps.txt`

---

## 4. 模型与 Prompt 映射表

| 模型 ID 模式 | Prompt 文件 | 核心特点 |
|-------------|-------------|----------|
| `claude*` | anthropic.txt | 任务管理、Task 工具 |
| `gpt-4`, `o1`, `o3` | beast.txt | 持续迭代、网络研究 |
| `gpt` (非 codex) | gpt.txt | 最小改动、双通道输出 |
| `gpt` + `codex` | codex.txt | 前端设计、文件引用 |
| `gemini-*` | gemini.txt | 规范遵循、新应用流程 |
| `trinity` | trinity.txt | 单工具调用 |
| `kimi` | kimi.txt | 中文支持、AGENTS.md |
| 其他 | default.txt | 通用简洁模式 |

---

## 5. 关键文件索引

| 文件 | 行数 | 说明 |
|------|------|------|
| `packages/opencode/src/session/system.ts` | 84 | 模型选择、environment/skills 构建 |
| `packages/opencode/src/session/prompt.ts` | 1908 | 完整 Prompt 构建逻辑 |
| `packages/opencode/src/session/prompt/*.txt` | - | 13 个内置 Prompt 模板 |

---

## 6. 总结

OpenCode 的 Prompt 系统设计:

- ✅ **分层组装**: 从模型选择 → 系统级 → 工具 → 上下文 → 模式特定
- ✅ **模型匹配**: 基于模型 ID 模式匹配不同 Prompt
- ✅ **动态扩展**: 支持 Skill 列表动态注入
- ✅ **模式切换**: 支持 plan/build/max-steps 模式切换
- ✅ **13 个内置 Prompt**: 覆盖主流模型和特殊场景

**核心文件**:
- `packages/opencode/src/session/system.ts` - 选择逻辑
- `packages/opencode/src/session/prompt.ts` - 组装逻辑
- `packages/opencode/src/session/prompt/*.txt` - 13 个模板