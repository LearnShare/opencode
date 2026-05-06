# Phase 4: Skill 系统 - 研究报告

> 研究时间: ~2 小时
> 状态: ✅ 已完成

## 1. Skill 核心概念

### 1.1 什么是 Skill

Skill 是 OpenCode 中的任务专用能力模块，允许用户定义可重用的技能。每个 Skill 是一个 **SKILL.md** 文件，包含:

- **name**: 唯一标识符
- **description**: 用途描述  
- **content**: 执行逻辑 (Markdown 格式)

### 1.2 Skill 定义格式

```yaml
---
name: browser-automation
description: Use browser automation for web interactions
---

# Skill 内容

This skill provides capabilities for browser automation...

## 工具

- 可以使用 Playwright 或 Selenium
- 支持网页截图
- 支持表单填写
```

### 1.3 核心数据结构

**文件**: `src/skill/index.ts` (297 行)

```typescript
export const Info = Schema.Struct({
  name: Schema.String,
  description: Schema.String,
  location: Schema.String,    // SKILL.md 文件路径
  content: Schema.String,    // 文件内容
})
export type Info = Schema.Schema.Type<typeof Info>

export interface Interface {
  readonly get: (name: string) => Effect.Effect<Info | undefined>
  readonly all: () => Effect.Effect<Info[]>
  readonly dirs: () => Effect.Effect<string[]>
  readonly available: (agent?: Agent.Info) => Effect.Effect<Info[]>
}
```

---

## 2. Skill 发现机制

### 2.1 搜索位置

**文件**: `src/skill/index.ts` (行 146-204)

| 来源 | 路径 | 说明 |
|------|------|------|
| **项目本地** | `skill/` 或 `skills/` | 项目内的 Skill |
| **全局目录** | `~/.claude/skills/` | Claude Code 兼容 |
| **全局目录** | `~/.agents/skills/` | Agents 目录 |
| **配置路径** | `opencode.json` 中的 `skills.paths` | 用户配置 |
| **远程** | `opencode.json` 中的 `skills.urls` | 远程 Skill 仓库 |

### 2.2 搜索模式

```typescript
// 本地模式
const OPENCODE_SKILL_PATTERN = "{skill,skills}/**/SKILL.md"

// 外部模式 (Claude/Agents)
const EXTERNAL_SKILL_PATTERN = "skills/**/SKILL.md"

// 通用模式
const SKILL_PATTERN = "**/SKILL.md"
```

### 2.3 远程 Skill 拉取

**文件**: `src/skill/discovery.ts` (116 行)

```typescript
// 从远程 URL 拉取 Skill
// 1. 获取 index.json
// 2. 解析 skill 列表
// 3. 下载每个 skill 的文件到缓存
// 4. 返回本地目录列表

const pull = Effect.fn("Discovery.pull")(function* (url: string) {
  const base = url.endsWith("/") ? url : `${url}/`
  const index = new URL("index.json", base).href
  
  // 获取 index.json
  const data = yield* HttpClientRequest.get(index).pipe(
    HttpClientRequest.acceptJson,
    http.execute,
    Effect.flatMap(HttpClientResponse.schemaBodyJson(Index)),
  )
  
  // 下载每个 skill 到缓存
  for (const skill of data.skills) {
    // 下载 skill.files 中的所有文件
    // 返回本地目录路径
  }
})
```

### 2.4 加载流程

```
发现阶段 (Discovery)
├── 扫描本地 skill/ 目录
├── 扫描全局 ~/.claude/skills/
├── 扫描配置路径
├── 拉取远程 Skill
└── 返回匹配文件列表

加载阶段 (Loading)
├── 解析每个 SKILL.md
├── 验证 frontmatter (name, description)
├── 存储到 State
└── 注册为工具
```

---

## 3. Skill 与工具集成

### 3.1 作为工具暴露

**文件**: `src/tool/skill.ts` (75 行)

```typescript
export const SkillTool = Tool.define(
  "skill",
  Effect.gen(function* () {
    const skill = yield* Skill.Service
    const rg = yield* Ripgrep.Service

    return {
      description: DESCRIPTION,
      parameters: Parameters,
      execute: (params, ctx) => Effect.gen(function* () {
        // 1. 获取 Skill 信息
        const info = yield* skill.get(params.name)
        if (!info) throw new Error(`Skill "${params.name}" not found`)
        
        // 2. 请求权限
        yield* ctx.ask({
          permission: "skill",
          patterns: [params.name],
          always: [params.name],
        })

        // 3. 获取相关文件
        const dir = path.dirname(info.location)
        const files = yield* rg.files({ cwd: dir, ... }).pipe(
          Stream.take(10),
          Stream.runCollect,
        )

        // 4. 返回 Skill 内容
        return {
          title: `Loaded skill: ${info.name}`,
          output: [
            `<skill_content name="${info.name}">`,
            `# Skill: ${info.name}`,
            info.content.trim(),
            `<skill_files>${files}</skill_files>`,
            "</skill_content>",
          ].join("\n"),
          metadata: { name: info.name, dir },
        }
      }),
    }
  }),
)
```

### 3.2 工具参数

```typescript
export const Parameters = Schema.Struct({
  name: Schema.String.annotate({ description: "The name of the skill from available_skills" }),
})
```

### 3.3 Skill 执行结果

当 LLM 调用 Skill 工具时:

1. **返回 Skill 内容**: Markdown 内容作为工具输出
2. **提供基础目录**: 告知 LLM 相关文件的相对路径
3. **文件列表**: 列出 Skill 目录中的相关文件 (最多 10 个)

---

## 4. 权限管理

### 4.1 Skill 权限评估

```typescript
// 在 available() 方法中
const available = Effect.fn("Skill.available")(function* (agent?: Agent.Info) {
  const s = yield* InstanceState.get(state)
  const list = Object.values(s.skills).toSorted(...)
  if (!agent) return list
  
  // 权限过滤
  return list.filter((skill) => 
    Permission.evaluate("skill", skill.name, agent.permission).action !== "deny"
  )
})
```

### 4.2 权限规则

- **allow**: 允许使用
- **deny**: 拒绝使用
- **ask**: 询问用户

---

## 5. 关键文件索引

| 文件 | 行数 | 说明 |
|------|------|------|
| `src/skill/index.ts` | 297 | Skill Service 主文件 |
| `src/skill/discovery.ts` | 116 | 远程 Skill 拉取 |
| `src/tool/skill.ts` | 75 | Skill 工具定义 |
| `src/config/skills.ts` | - | Skill 配置结构 |

---

## 6. 待深入研究的问题

1. **Skill 模板化**: 是否有内置的 Skill 模板?
2. **Skill 版本**: 如何处理 Skill 更新?
3. **Skill 组合**: Skill 之间如何相互调用?

---

## 7. 总结

Phase 4 完成了对 Skill 系统的深入理解:

- ✅ 理解了 Skill 定义和结构 (SKILL.md)
- ✅ 理解了 Skill 发现机制 (本地/全局/远程)
- ✅ 理解了 Skill 与工具的集成 (作为 Tool 暴露)
- ✅ 理解了 Skill 权限管理

**关键发现**:
- Skill 是 Markdown 文件，通过 frontmatter 定义元数据
- 支持从远程仓库拉取 Skill (通过 index.json)
- Skill 作为工具暴露给 LLM，内容作为上下文
- 与 Permission 系统集成进行权限控制

**下一步**: Phase 5 - MCP 集成