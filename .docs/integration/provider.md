# Phase 3: Provider - LLM 提供商 - 研究报告

> 研究时间: ~4 小时
> 状态: ✅ 已完成

## 1. Provider 接口和架构

### 1.1 核心组件

**文件**: `src/provider/provider.ts` (1756 行)

```typescript
export interface Interface {
  readonly list: () => Effect.Effect<Record<ProviderID, Info>>
  readonly getProvider: (providerID: ProviderID) => Effect.Effect<Info>
  readonly getModel: (providerID: ProviderID, modelID: ModelID) => Effect.Effect<Model>
  readonly getLanguage: (model: Model) => Effect.Effect<LanguageModelV3>
  readonly closest: (providerID: ProviderID, query: string[]) => Effect.Effect<{ providerID; modelID } | undefined>
  readonly getSmallModel: (providerID: ProviderID) => Effect.Effect<Model | undefined>
  readonly defaultModel: () => Effect.Effect<{ providerID: ProviderID; modelID: ModelID }>
}
```

### 1.2 数据模型

**Provider 信息** (`src/provider/schema.ts`):

```typescript
// ProviderID - 品牌化字符串
export type ProviderID = typeof providerIdSchema.Type
// 预定义 Provider
ProviderID.opencode    // OpenCode 内置
ProviderID.anthropic   // Anthropic (Claude)
ProviderID.openai      // OpenAI (GPT)
ProviderID.google      // Google (Gemini)
ProviderID.azure       // Azure OpenAI
ProviderID.amazonBedrock // AWS Bedrock
// ...

// ModelID - 品牌化字符串
export type ModelID = typeof modelIdSchema.Type
```

**Provider 详情**:

```typescript
export const Info = Schema.Struct({
  id: ProviderID,
  name: Schema.String,
  source: Schema.Literals(["env", "config", "custom", "api"]),  // 来源优先级
  env: Schema.Array(Schema.String),  // 环境变量名
  key: optionalOmitUndefined(Schema.String),
  options: Schema.Record(Schema.String, Schema.Any),  // SDK 选项
  models: Schema.Record(Schema.String, Model),  // 模型列表
})
```

**Model 详情**:

```typescript
export const Model = Schema.Struct({
  id: ModelID,
  providerID: ProviderID,
  api: ProviderApiInfo,  // { id, url, npm }
  name: Schema.String,
  family: optionalOmitUndefined(Schema.String),
  capabilities: ProviderCapabilities,  // 支持的功能
  cost: ProviderCost,  // 价格
  limit: ProviderLimit,  // 上下文限制
  status: Schema.Literals(["alpha", "beta", "deprecated", "active"]),
  options: Schema.Record(Schema.String, Schema.Any),
  headers: Schema.Record(Schema.String, Schema.String),
  release_date: Schema.String,
  variants: optionalOmitUndefined(Schema.Record(...)),  // 模型变体
})
```

### 1.3 Provider 优先级

Provider 来源优先级 (从低到高):

```
1. custom  - models.dev 内置定义
2. env      - 环境变量检测
3. api      - 认证存储 (API Key)
4. config   - opencode.json 配置 (最高优先级)
```

### 1.4 内部状态

```typescript
interface State {
  models: Map<string, LanguageModelV3>      // 缓存的 LanguageModel 实例
  providers: Record<ProviderID, Info>       // Provider 详情
  sdk: Map<string, BundledSDK>              // SDK 实例缓存
  modelLoaders: Record<string, CustomModelLoader>   // 模型加载器
  varsLoaders: Record<string, CustomVarsLoader>      // 环境变量加载器
}
```

---

## 2. 支持的模型列表

### 2.1 内置 SDK

**文件**: `src/provider/provider.ts` (行 92-117)

**Bundled Providers** (通过 `@ai-sdk/*` 包):

| SDK 包 | Provider ID |
|--------|-------------|
| `@ai-sdk/anthropic` | anthropic |
| `@ai-sdk/openai` | openai |
| `@ai-sdk/google` | google |
| `@ai-sdk/google-vertex` | google-vertex |
| `@ai-sdk/google-vertex/anthropic` | google-vertex-anthropic |
| `@ai-sdk/azure` | azure |
| `@ai-sdk/amazon-bedrock` | amazon-bedrock |
| `@ai-sdk/mistral` | mistral |
| `@ai-sdk/groq` | groq |
| `@ai-sdk/deepinfra` | deepinfra |
| `@ai-sdk/cerebras` | cerebras |
| `@ai-sdk/cohere` | cohere |
| `@ai-sdk/perplexity` | perplexity |
| `@ai-sdk/vercel` | vercel |
| `@ai-sdk/xai` | xai |
| `@ai-sdk/alibaba` | alibaba |
| `@ai-sdk/gateway` | gateway |
| `@ai-sdk/togetherai` | togetherai |
| `@openrouter/ai-sdk-provider` | openrouter |
| `gitlab-ai-provider` | gitlab |
| `@ai-sdk/github-copilot` | github-copilot |
| `venice-ai-sdk-provider` | venice |

### 2.2 自定义加载

**文件**: `src/provider/provider.ts` (行 149-833)

```typescript
function custom(dep: CustomDep): Record<string, CustomLoader> {
  return {
    // Anthropic - 使用 beta headers
    anthropic: () => Effect.succeed({
      autoload: false,
      options: { headers: { "anthropic-beta": "..." } }
    }),

    // OpenCode - 特殊处理免费模型
    opencode: Effect.fnUntraced(function* (input) {
      // 检查环境变量、认证、配置
      // 移除非免费模型如果未认证
    }),

    // OpenAI - 使用 Responses API
    openai: () => Effect.succeed({
      autoload: false,
      async getModel(sdk, modelID) { return sdk.responses(modelID) }
    }),

    // GitHub Copilot - 智能选择 API
    "github-copilot": () => Effect.succeed({
      autoload: false,
      async getModel(sdk, modelID) {
        if (useLanguageModel(sdk)) return sdk.languageModel(modelID)
        return shouldUseCopilotResponsesApi(modelID) 
          ? sdk.responses(modelID) 
          : sdk.chat(modelID)
      }
    }),

    // Azure - 特殊模型选择逻辑
    azure: Effect.fnUntraced(function* (provider) {...}),

    // Amazon Bedrock - 区域前缀处理
    "amazon-bedrock": Effect.fnUntraced(function* () {...}),

    // Google Vertex - 项目和位置配置
    "google-vertex": Effect.fnUntraced(function* (provider) {...}),

    // Cloudflare Workers AI
    "cloudflare-workers-ai": Effect.fnUntraced(function* (input) {...}),

    // GitLab - 支持 workflow 模型发现
    gitlab: Effect.fnUntraced(function* (input) {...}),

    // ... 更多
  }
}
```

### 2.3 模型能力

```typescript
const ProviderCapabilities = Schema.Struct({
  temperature: Schema.Boolean,    // 温度控制
  reasoning: Schema.Boolean,      // 推理能力
  attachment: Schema.Boolean,     // 附件支持
  toolcall: Schema.Boolean,       // 工具调用
  input: ProviderModalities,      // 输入模态 (text/audio/image/video/pdf)
  output: ProviderModalities,     // 输出模态
  interleaved: ProviderInterleaved,  // 交错推理
})
```

### 2.4 模型变体 (Variants)

**文件**: `src/provider/transform.ts` (1309 行)

```typescript
// 变体机制允许同一基础模型有不同的配置
// 例如: claude-3-5-sonnet-20241022-fast
//       claude-3-5-sonnet-20241022-writing
```

---

## 3. 认证机制

### 3.1 认证类型

**文件**: `src/provider/auth.ts` (228 行)

```typescript
export class Method extends Schema.Class<Method>("ProviderAuthMethod")({
  type: Schema.Literals(["oauth", "api"]),  // 认证类型
  label: Schema.String,                      // 显示名称
  prompts: optionalOmitUndefined(Schema.Array(Prompt)),  // 交互式输入
})

// Prompt 类型
const TextPrompt = Schema.Struct({
  type: Schema.Literal("text"),
  key: Schema.String,
  message: Schema.String,
  placeholder: optionalOmitUndefined(Schema.String),
})

const SelectPrompt = Schema.Struct({
  type: Schema.Literal("select"),
  key: Schema.String,
  options: Schema.Array(SelectOption),
})
```

### 3.2 认证来源

1. **环境变量**: 检查 `ANTHROPIC_API_KEY`, `OPENAI_API_KEY` 等
2. **配置**: `opencode.json` 中的 `provider` 配置
3. **API Key 存储**: 通过 `auth` 命令存储的密钥
4. **OAuth**: 支持 OAuth 流程的 Provider

### 3.3 认证接口

**文件**: `src/auth/` - 通用认证系统

```typescript
// Auth Service
export interface Interface {
  readonly get: (id: string) => Effect.Effect<Auth.Info | undefined>
  readonly all: () => Effect.Effect<Record<string, Auth.Info>>
  readonly set: (id: string, info: Auth.Info) => Effect.Effect<void>
  readonly remove: (id: string) => Effect.Effect<void>
}
```

---

## 4. 模型选择逻辑

### 4.1 默认模型选择

```typescript
// 优先级
1. 用户指定模型 (session 配置)
2. Agent 默认模型
3. Provider 默认模型 (第一个模型)
4. 全局默认模型
```

### 4.2 上下文窗口处理

```typescript
// 根据模型限制截断上下文
const limit = model.limit.context
// 自动压缩会话如果超出限制
```

### 4.3 小模型选择

```typescript
// 获取支持的小型模型 (用于快速任务)
getSmallModel(providerID: ProviderID): Effect.Effect<Model | undefined>
```

### 4.4 模糊搜索

```typescript
// 通过名称搜索模型
closest(providerID: ProviderID, query: string[]): Effect.Effect<{ providerID, modelID } | undefined>
```

---

## 5. 关键文件索引

| 文件 | 行数 | 说明 |
|------|------|------|
| `src/provider/provider.ts` | 1756 | 核心 Provider Service |
| `src/provider/schema.ts` | 36 | ProviderID/ModelID 类型 |
| `src/provider/models.ts` | - | 内置模型定义 |
| `src/provider/transform.ts` | 1309 | 模型转换和变体 |
| `src/provider/auth.ts` | 228 | Provider 认证 |
| `src/provider/error.ts` | - | 错误类型定义 |
| `src/auth/index.ts` | - | 通用认证系统 |

---

## 6. 待深入研究的问题

1. **模型发现**: GitLab workflow 模型动态发现机制
2. **自定义 Provider**: 如何添加新的 Provider
3. **成本计算**: 实际使用成本如何计算
4. **模型切换**: 运行时模型切换的处理

---

## 7. 总结

Phase 3 完成了对 Provider 系统的深入理解:

- ✅ 理解了 Provider 接口和数据模型
- ✅ 理解了支持的模型和 SDK (20+ 内置 Provider)
- ✅ 理解了认证机制 (环境变量/API Key/OAuth)
- ✅ 理解了模型选择和优先级逻辑

**下一步**: Phase 4 - Skill 系统