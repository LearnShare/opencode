# Token 消耗分析与优化策略

> 研究时间: ~3 小时
> 状态: ✅ 已完成

---

## 1. Token 消耗构成分析

### 1.1 总体分布

在多轮对话中，Token 消耗主要集中在以下几个部分:

| 组成部分 | 占比估计 | 说明 |
|----------|---------|------|
| **Tool Output (工具输出)** | 40-60% | 历史工具执行结果，是最大的消耗来源 |
| **对话历史 (Messages)** | 20-30% | user/assistant/text 消息内容 |
| **System Prompt** | 5-15% | 环境信息、指令、技能描述 |
| **Tool Definitions** | 5-10% | 工具定义 (JSON Schema) |
| **其他** | <5% | 推理、缓存等 |

---

## 2. 各部分详细分析

### 2.1 Tool Output (工具输出) - 最大消耗

**文件**: `compaction.ts:300-344`

```typescript
// 压缩时的裁剪逻辑
const PRUNE_PROTECT = 40_000    // 保护阈值: 40k tokens
const PRUNE_MINIMUM = 20_000    // 最小裁剪量: 20k tokens
const TOOL_OUTPUT_MAX_CHARS = 2_000  // 最大保留字符: 2000
```

**现状**:
- 默认保留每个工具输出 **2000 字符** (约 500 tokens)
- 超过 40k tokens 的历史工具输出会被裁剪
- 保留最近 **2 轮** 完整对话

**问题**:
```typescript
// 示例: 一个大型项目搜索
Glob 结果: 可能返回 100+ 文件路径 → 3000+ 字符
Grep 结果: 大量匹配行 → 10000+ 字符
Shell 输出: 编译错误日志 → 5000+ 字符
```

### 2.2 对话历史

**文件**: `message-v2.ts:791-979`

每轮累积的消息结构:
```typescript
// 第一轮: 约 100-500 tokens
{ role: "user", content: "帮我修复这个 bug" }

// 第二轮: 约 500-2000 tokens
{ role: "assistant", content: "我来分析..." },
{ role: "tool", content: "文件内容..." },
{ role: "assistant", content: "找到问题了..." },
{ role: "tool", content: "修改完成" }

// 第 N 轮: 线性增长
// 每轮大约增加 200-1000 tokens (取决于工具输出)
```

### 2.3 System Prompt

**文件**: `system.ts:48-62`

```typescript
environment(model) = [
  `You are powered by the model named ${model.api.id}`,
  `Working directory: ${ctx.directory}`,
  `Workspace root folder: ${ctx.worktree}`,
  `Platform: ${process.platform}`,
  `Today's date: ...`,
]
// 约 200-500 tokens
```

**技能描述** (`system.ts:65-77`):
```typescript
skills(agent) = [
  "Skills provide specialized instructions...",
  Skill.fmt(list, { verbose: true }),
]
// 动态生成，取决于可用技能数量
```

### 2.4 Tool Definitions

**文件**: `prompt.ts:415-456`

每个工具的 JSON Schema:
```typescript
{
  name: "Read",
  description: "Read the full contents of a file...",
  parameters: {
    type: "object",
    properties: {
      filePath: { type: "string", description: "..." }
    },
    required: ["filePath"]
  }
}
// 每个工具约 100-300 tokens
// 10 个工具约 1000-3000 tokens
```

---

## 3. 当前优化机制

### 3.1 工具输出裁剪 (Pruning)

**文件**: `compaction.ts:300-344`

```typescript
const prune = Effect.fn("SessionCompaction.prune")(function* (input: { sessionID: SessionID }) {
  // 1. 从最旧的消息开始
  // 2. 保留最近 2 轮对话
  // 3. 遇到 summary 消息停止
  // 4. 累积超过 40k tokens 后标记待裁剪
  // 5. 总裁剪量超过 20k 时执行
})
```

**配置项** (`config.ts:227-246`):
```typescript
compaction: {
  auto: true,                    // 自动压缩
  prune: true,                   // 裁剪旧工具输出
  tail_turns: 2,                // 保留最近 2 轮
  preserve_recent_tokens: 8000,  // 保留 8000 tokens
  reserved: 20000,               // 预留缓冲
}
```

### 3.2 工具输出截断

**文件**: `message-v2.ts:326-330`

```typescript
function truncateToolOutput(text: string, maxChars?: number) {
  if (!maxChars || text.length <= maxChars) return text
  const omitted = text.length - maxChars
  return `${text.slice(0, maxChars)}\n[Tool output truncated: omitted ${omitted} chars]`
}
```

### 3.3 压缩机制

**触发条件**: `tokens.total >= usable(context)`

**保留策略**:
- 默认保留最近 **2 轮** 完整对话
- 保留 **8000 tokens** (或 25% 可用上下文)
- 之前的历史被压缩为摘要

---

## 4. 优化策略建议

### 4.1 高优先级优化

#### 4.1.1 智能工具输出截断

**问题**: 当前固定 2000 字符截断，不区分工具类型

**优化方案**:
```typescript
// 根据工具类型设置不同的截断长度
const TOOL_OUTPUT_LIMITS = {
  // 高价值 - 保留更多
  Read: 5000,        // 文件内容可能重要
  Grep: 3000,        // 搜索结果

  // 中等价值 - 适度保留
  Glob: 1000,        // 文件列表
  Edit: 1000,        // 编辑结果

  // 低价值 - 快速截断
  Shell: 1500,       // 命令输出
  Task: 500,         // 子任务结果
}
```

#### 4.1.2 按工具类型选择保留策略

**问题**: 所有工具输出同等对待

**优化方案**:
```typescript
// 保护特定工具的完整输出
const PRUNE_PROTECTED_TOOLS = [
  "skill",           // Skill 执行结果
  "mcp",            // MCP 工具
  "Read",           // 重要文件内容
]

// 其他工具可以裁剪
```

### 4.2 中优先级优化

#### 4.2.1 动态 System Prompt

**问题**: 每轮重复发送相同的系统提示

**优化方案**:
- 实现 Prompt Cache (如果模型支持)
- 减少冗余环境信息
- 只在环境变化时更新

#### 4.2.2 消息压缩优化

**当前**:
- 固定保留 2 轮
- 固定保留 8000 tokens

**优化方案**:
- 根据任务类型动态调整
- 代码编辑任务: 保留更多上下文
- 简单查询任务: 快速压缩

### 4.3 低优先级优化

#### 4.3.1 工具定义优化

```typescript
// 简化工具描述
const tools = {
  // 当前: 详细描述
  Read: {
    description: "Read the full contents of a file at the given path. Returns the file content as a string.",
    parameters: { ... }
  }

  // 优化: 简洁描述
  Read: {
    description: "Read file",
    parameters: { ... }
  }
}
```

#### 4.3.2 推理token优化

对于支持推理的模型 (如 o1, reasoning):
- 限制推理 token 数量
- 不保存推理过程到历史

---

## 5. 具体配置建议

### 5.1 配置文件 (.opencode.yaml)

```yaml
compaction:
  # 启用自动压缩
  auto: true

  # 启用工具输出裁剪
  prune: true

  # 保留最近 3 轮完整对话
  tail_turns: 3

  # 保留 10000 tokens
  preserve_recent_tokens: 10000

  # 预留 15000 缓冲
  reserved: 15000

# 工具输出截断配置
truncation:
  Read: 5000
  Grep: 3000
  Glob: 1000
  Shell: 1500
  Edit: 1000
  default: 2000
```

### 5.2 场景优化

#### 大项目场景
```yaml
compaction:
  tail_turns: 4
  preserve_recent_tokens: 12000
truncation:
  Read: 8000  # 需要更多上下文
```

#### 简单任务场景
```yaml
compaction:
  tail_turns: 1
  preserve_recent_tokens: 4000
truncation:
  default: 1000
```

---

## 6. 总结

### Token 消耗占比

| 组件 | 占比 | 优化难度 |
|------|------|---------|
| Tool Output | 40-60% | ⭐⭐ 简单 - 调整截断长度 |
| 对话历史 | 20-30% | ⭐⭐⭐ 中等 - 改进压缩算法 |
| System | 5-15% | ⭐ 简单 - 精简描述 |
| Tool Defs | 5-10% | ⭐ 简单 - 简化 schema |

### 核心优化方向

1. **工具输出截断** - 最有效的优化点
2. **压缩保留策略** - 根据任务类型动态调整
3. **System Prompt 精简** - 去除冗余信息
4. **工具定义精简** - 简化 JSON Schema

---

## 7. 关键文件索引

| 文件 | 说明 |
|------|------|
| `src/session/compaction.ts` | 压缩和裁剪逻辑 |
| `src/session/message-v2.ts` | 工具输出截断 |
| `src/config/config.ts` | 压缩配置项 |
| `src/session/system.ts` | System Prompt 构建 |
| `src/session/prompt.ts` | 工具定义构建 |
| `src/util/token.ts` | Token 估算 |