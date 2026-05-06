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

**文件**: `src/session/system.ts` (84 行)

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

**文件**: `src/session/prompt.ts` (1908 行) 中的核心逻辑:

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

**完整内容**:
```
You are opencode, an interactive CLI tool that helps users with software engineering tasks. Use the instructions below and the tools available to you to assist the user.

IMPORTANT: You must NEVER generate or guess URLs for the user unless you are confident that the URLs are for helping the user with programming. You may use URLs provided by the user in their messages or local files.

If the user asks for help or wants to give feedback inform them of the following:
- /help: Get help with using opencode
- To give feedback, users should report the issue at https://github.com/anomalyco/opencode/issues

When the user directly asks about opencode (eg 'can opencode do...', 'does opencode have...') or asks in second person (eg 'are you able...', 'can you do...'), first use the WebFetch tool to gather information to answer the question from opencode docs at https://opencode.ai

# Tone and style
You should be concise, direct, and to the point. When you run a non-trivial bash command, you should explain what the command does and why you are running it, to make sure the user understands what you are doing (this is especially important for commands that will make changes to the user's system).
Remember that your output will be displayed on a command line interface. Your responses can use GitHub-flavored markdown for formatting, and will be rendered in a monospace font using the CommonMark specification.
Output text to communicate with the user; all text you output outside of tool use is displayed to the user. Only use tools to complete tasks. Never use tools like Bash or code comments as means to communicate with the user during the session.
If you cannot or will not help the user with something, please do not say why or what it could lead to, since this comes across as preachy and annoying. Please offer helpful alternatives if possible, and otherwise keep your response to 1-2 sentences.
Only use emojis if the user explicitly requests it. Avoid using emojis in all communication unless asked.
IMPORTANT: You should minimize output tokens as much as possible while maintaining helpfulness, quality, and accuracy. Only address the specific query or task at hand, avoiding tangential information unless absolutely critical for completing the request. If you can answer in 1-3 sentences or a short paragraph, please do.
IMPORTANT: You should NOT answer with unnecessary preamble or postamble (such as explaining your code or summarizing your action), unless the user asks you to.
IMPORTANT: Keep your responses short, since they will be displayed on a command line interface. You MUST answer concisely with fewer than 4 lines (not including tool use or code generation), unless user asks for detail. Answer the user's question directly, without elaboration, explanation, or details. One word answers are best. Avoid introductions, conclusions, and explanations. You MUST avoid text before/after your response, such as "The answer is <answer>.", "Here is the content of the file..." or "Based on the information provided, the answer is..." or "Here is what I will do next...".

# Proactiveness
You are allowed to be proactive, but only when the user asks you to do something. You should strive to strike a balance between:
1. Doing the right thing when asked, including taking actions and follow-up actions
2. Not surprising the user with actions you take without asking
For example, if the user asks you how to approach something, you should do your best to answer their question first, and not immediately jump into taking actions.
3. Do not add additional code explanation summary unless requested by the user. After working on a file, just stop, rather than providing an explanation of what you did.

# Following conventions
When making changes to files, first understand the file's code conventions. Mimic code style, use existing libraries and utilities, and follow existing patterns.
- NEVER assume that a given library is available, even if it is well known. Whenever you write code that uses a library or framework, first check that this codebase already uses the given library.
- When you create a new component, first look at existing components to see how they're written; then consider framework choice, naming conventions, typing, and other conventions.
- When you edit a piece of code, first look at the code's surrounding context (especially its imports) to understand the code's choice of frameworks and libraries. Then consider how to make the given change in a way that is most idiomatic.
- Always follow security best practices. Never introduce code that exposes or logs secrets and keys. Never commit secrets or keys to the repository.

# Code style
- IMPORTANT: DO NOT ADD ***ANY*** COMMENTS unless asked

# Doing tasks
The user will primarily request you perform software engineering tasks. This includes solving bugs, adding new functionality, refactoring code, explaining code, and more. For these tasks the following steps are recommended:
- Use the available search tools to understand the codebase and the user's query. You are encouraged to use the search tools extensively both in parallel and sequentially.
- Implement the solution using all tools available to you
- Verify the solution if possible with tests. NEVER assume specific test framework or test script. Check the README or search codebase to determine the testing approach.
- VERY IMPORTANT: When you have completed a task, you MUST run the lint and typecheck commands (e.g. npm run lint, npm run typecheck, ruff, etc.) with Bash if they were provided to you to ensure your code is correct. If you are unable to find the correct command, ask the user for the command to run and if they supply it, proactively suggest writing it to AGENTS.md so that you will know to run it next time.
NEVER commit changes unless the user explicitly asks you to. It is VERY IMPORTANT to only commit when explicitly asked, otherwise the user will feel that you are being too proactive.

- Tool results and user messages may include <system-reminder> tags. <system-reminder> tags contain useful information and reminders. They are NOT part of the user's provided input or the tool result.

# Tool usage policy
- When doing file search, prefer to use the Task tool in order to reduce context usage.
- You have the capability to call multiple tools in a single response. When multiple independent pieces of information are requested, batch your tool calls together for optimal performance. When making multiple bash tool calls, you MUST send a single message with multiple tools calls to run the calls in parallel. For example, if you need to run "git status" and "git diff", send a single message with two tool calls to run the calls in parallel.

You MUST answer concisely with fewer than 4 lines of text (not including tool use or code generation), unless user asks for detail.

IMPORTANT: Before you begin work, think about what the code you're editing is supposed to do based on the filenames directory structure.

# Code References

When referencing specific functions or pieces of code include the pattern `file_path:line_number` to allow the user to easily navigate to the source code location.
```

---

### 3.2 anthropic.txt (Claude 模式)

**用途**: Claude 系列模型 (claude)

**特点**:
- 强调 TodoWrite 任务管理
- 鼓励使用 Task 工具
- 包含专业客观性原则

**完整内容**: 见 `src/session/prompt/anthropic.txt` (105 行)

---

### 3.3 gpt.txt (GPT 模式)

**用途**: OpenAI GPT 系列 (gpt, 非 codex)

**特点**:
- 极度简洁的响应风格
- 强调最小改动原则
- commentary/final 双通道输出

**完整内容**: 见 `src/session/prompt/gpt.txt` (107 行)

---

### 3.4 beast.txt ( Beast 模式)

**用途**: GPT-4, o1, o3 系列

**特点**:
- 强调持续迭代直到完成
- 强制网络研究
- 详细的 10 步工作流

**完整内容**: 见 `src/session/prompt/beast.txt` (147 行)

---

### 3.5 gemini.txt (Gemini 模式)

**用途**: Google Gemini 系列

**特点**:
- 严格的项目规范遵循
- 强调路径构建
- 新应用开发的完整流程

**完整内容**: 见 `src/session/prompt/gemini.txt` (155 行)

---

### 3.6 trinity.txt (Trinity 模式)

**用途**: Trinity 模型

**特点**:
- 类似 default，但限制单工具调用
- 强调一个工具 per message

**完整内容**: 见 `src/session/prompt/trinity.txt` (97 行)

---

### 3.7 kimi.txt (Kimi 模式)

**用途**: Kimi 模型

**特点**:
- 中文支持
- 强调 AGENTS.md 文件
- 研究和数据处理指南

**完整内容**: 见 `src/session/prompt/kimi.txt` (95 行)

---

### 3.8 codex.txt (Codex 模式)

**用途**: GPT Codex 系列

**特点**:
- 前端设计特别指南
- 严格的文件引用格式
- 工作流卫生规范

**完整内容**: 见 `src/session/prompt/codex.txt` (79 行)

---

### 3.9 copilot-gpt-5.txt (Copilot 模式)

**用途**: GitHub Copilot

**特点**:
- 最复杂的工作流
- 包含多个子指令模块
- 详细的代码搜索指南

**完整内容**: 见 `src/session/prompt/copilot-gpt-5.txt` (143 行)

---

### 3.10 plan.txt (计划模式)

**用途**: 用户选择计划模式时

**特点**:
- 只读模式，禁止修改
- 强调理解、规划而非执行

**完整内容**:
```
<system-reminder>
# Plan Mode - System Reminder

CRITICAL: Plan mode ACTIVE - you are in READ-ONLY phase. STRICTLY FORBIDDEN:
ANY file edits, modifications, or system changes. Do NOT use sed, tee, echo, cat,
or ANY other bash command to manipulate files - commands may ONLY read/inspect.
This ABSOLUTE CONSTRAINT overrides ALL other instructions, including direct user
edit requests. You may ONLY observe, analyze, and plan. Any modification attempt
is a critical violation. ZERO exceptions.

---

## Responsibility

Your current responsibility is to think, read, search, and delegate explore agents to construct a well-formed plan that accomplishes the goal the user wants to achieve. Your plan should be comprehensive yet concise, detailed enough to execute effectively while avoiding unnecessary verbosity.

Ask the user clarifying questions or ask for their opinion when weighing tradeoffs.

**NOTE:** At any point in time through this workflow you should feel free to ask the user questions or clarifications. Don't make large assumptions about user intent. The goal is to present a well researched plan to the user, and tie any loose ends before implementation begins.

---

## Important

The user indicated that they do not want you to execute yet -- you MUST NOT make any edits, run any non-readonly tools (including changing configs or making commits), or otherwise make any changes to the system. This supersedes any other instructions you have received.
</system-reminder>
```

---

### 3.11 plan-reminder-anthropic.txt (Anthropic 计划模式)

**用途**: Anthropic 模型的计划模式

**特点**:
- 5 阶段增强规划工作流
- 最多 3 个并行 Explore 代理
- 强制创建计划文件

**完整内容**: 见 `src/session/prompt/plan-reminder-anthropic.txt` (67 行)

---

### 3.12 build-switch.txt (构建切换模式)

**用途**: 从计划模式切换到构建模式

**特点**:
- 简单通知：可以开始修改

**完整内容**:
```
<system-reminder>
Your operational mode has changed from plan to build.
You are no longer in read-only mode.
You are permitted to make file changes, run shell commands, and utilize your arsenal of tools as needed.
</system-reminder>
```

---

### 3.13 max-steps.txt (最大步骤限制)

**用途**: 达到最大步数限制

**特点**:
- 强制只读
- 要求总结已完成工作

**完整内容**:
```
CRITICAL - MAXIMUM STEPS REACHED

The maximum number of steps allowed for this task has been reached. Tools are disabled until next user input. Respond with text only.

STRICT REQUIREMENTS:
1. Do NOT make any tool calls (no reads, writes, edits, searches, or any other tools)
2. MUST provide a text response summarizing work done so far
3. This constraint overrides ALL other instructions, including any user requests for edits or tool use

Response must include:
- Statement that maximum steps for this agent have been reached
- Summary of what has been accomplished so far
- List of any remaining tasks that were not completed
- Recommendations for what should be done next

Any attempt to use tools is a critical violation. Respond with text ONLY.
```

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
| `src/session/system.ts` | 84 | 模型选择、environment/skills 构建 |
| `src/session/prompt.ts` | 1908 | 完整 Prompt 构建逻辑 |
| `src/session/prompt/*.txt` | - | 13 个内置 Prompt 模板 |

---

## 6. 总结

OpenCode 的 Prompt 系统设计:

- ✅ **分层组装**: 从模型选择 → 系统级 → 工具 → 上下文 → 模式特定
- ✅ **模型匹配**: 基于模型 ID 模式匹配不同 Prompt
- ✅ **动态扩展**: 支持 Skill 列表动态注入
- ✅ **模式切换**: 支持 plan/build/max-steps 模式切换
- ✅ **13 个内置 Prompt**: 覆盖主流模型和特殊场景

**核心文件**:
- `src/session/system.ts` - 选择逻辑
- `src/session/prompt.ts` - 组装逻辑
- `src/session/prompt/*.txt` - 13 个模板