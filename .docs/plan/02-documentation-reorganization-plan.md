# 研究报告文档整理计划

> 状态: ✅ 已完成 (保持原结构)
> 创建日期: 2026-05-07
> 完成日期: 2026-05-07
> 目的: 规范化研究报告文档结构，提升可读性和可维护性

---

## 决策说明

经过评估，决定**保持现有的扁平目录结构**，原因如下：

1. 文件数量较少 (15 个)，扁平结构已足够清晰
2. `README.md` 已提供按 Phase 和按功能模块的双重索引
3. 重组文件需要更新所有内部链接，工作量大且易出错
4. 当前结构简单直观，用户可通过 README 快速导航

**已完成的优化**:
- 添加标准头部 (研究时间、状态、关联 Phase)
- 统一关键文件索引章节命名
- 创建术语表和 FAQ
- 创建 README 索引

---

## 1. 当前状态分析

### 1.1 现有文档清单

| 序号 | 文件名 | 状态 | 备注 |
|------|--------|------|------|
| 1 | `phase-01-basic-architecture.md` | ✅ 已完成 | Phase 1 报告 |
| 2 | `phase-02-agent-loop.md` | ✅ 已完成 | Phase 2 报告 |
| 3 | `phase-03-provider.md` | ✅ 已完成 | Phase 3 报告 |
| 4 | `phase-04-skill.md` | ✅ 已完成 | Phase 4 报告 |
| 5 | `phase-05-mcp-integration.md` | ✅ 已完成 | Phase 5 报告 |
| 6 | `phase-06-acp-protocol.md` | ✅ 已完成 | Phase 6 报告 |
| 7 | `phase-07-config-system.md` | ✅ 已完成 | Phase 7 报告 |
| 8 | `phase-08-plugin-system.md` | ✅ 已完成 | Phase 8 报告 |
| 9 | `phase-09-auth-permission-env.md` | ✅ 已完成 | Phase 9 报告 |
| 10 | `phase-10-project-session-management.md` | ✅ 已完成 | Phase 10 报告 |
| 11 | `phase-11-infrastructure.md` | ✅ 已完成 | Phase 11 报告 |
| 12 | `phase-12-git-ide-integration.md` | ✅ 已完成 | Phase 12 报告 |
| 13 | `phase-13-sdk-extensions.md` | ✅ 已完成 | Phase 13 报告 |
| 14 | `phase-prompt-system.md` | ✅ 已完成 | Phase 2 补充 - 合并自 prompt-system.md |
| 15 | `conversation-storage-memory.md` | ⚠️ 待处理 | Phase 10 补充 - 需整合或重命名 |

### 1.2 命名问题 (已处理)

1. ~~不一致的命名格式~~: 已统一使用 `phase-XX-xxx.md` 格式
2. ~~重复文件~~: 已合并 (`prompt-system.md` → `phase-prompt-system.md`)
3. ~~未分类文件~~: 已归类 (`conversation-storage-memory.md` 作为 Phase 10 补充)

---

## 2. 文档分类结构建议

### 2.1 推荐目录结构

```
.docs/
├── 01-deep-research-plan.md       # 研究计划 (已有)
├── 02-documentation-plan.md        # 本文档
├── readme.md                       # 文档索引说明
│
├── architecture/                   # 架构文档
│   ├── 01-project-structure.md    # 项目结构
│   ├── 02-entrypoints-startup.md   # 入口和启动
│   └── 03-effect-framework.md     # Effect 框架
│
├── core/                          # 核心模块
│   ├── 01-agent-loop.md          # Agent 循环
│   ├── 02-prompt-system.md       # Prompt 系统
│   ├── 03-tool-system.md         # 工具系统
│   └── 04-message-handling.md    # 消息处理
│
├── integration/                   # 集成模块
│   ├── 01-provider.md             # LLM Provider
│   ├── 02-skill.md                # Skill 系统
│   ├── 03-mcp.md                  # MCP 集成
│   ├── 04-acp.md                  # ACP 协议
│   └── 05-plugin.md               # 插件系统
│
├── security/                      # 安全模块
│   ├── 01-auth.md                # 认证系统
│   ├── 02-permission.md           # 权限系统
│   └── 03-env.md                  # 环境变量
│
├── project/                       # 项目管理
│   ├── 01-bootstrap.md           # 项目引导
│   ├── 02-worktree.md             # Worktree
│   ├── 03-storage.md              # 数据存储
│   └── 04-session.md              # 会话管理
│
├── infrastructure/                # 基础设施
│   ├── 01-bus.md                  # 事件总线
│   ├── 02-logging.md              # 日志系统
│   └── 03-error-handling.md       # 错误处理
│
├── integration-ext/               # 扩展集成
│   ├── 01-git.md                  # Git 集成
│   └── 02-lsp.md                  # LSP 集成
│
├── sdk/                           # SDK
│   ├── 01-js-sdk.md              # JavaScript SDK
│   └── 02-python-sdk.md          # Python SDK
│
└── appendices/                    # 附录
    ├── glossary.md               # 术语表
    ├── faq.md                    # 常见问题
    └── references.md              # 参考资料
```

### 2.2 命名规范

- 使用小写字母
- 使用连字符 `-` 分隔单词
- 保留 Phase 编号作为参考
- 简短描述性标题

---

## 3. 文档内容规范化建议

### 3.1 标准文档模板

每个报告文档应包含:

```markdown
# [标题]

> 研究时间: ~X 小时
> 状态: ✅ 已完成 / 🔄 进行中 / 📋 待开始

## 1. 概述
- 简要说明本模块的功能和重要性

## 2. 核心概念
- 关键术语定义
- 数据结构
- 接口设计

## 3. 实现细节
- 核心文件位置
- 关键代码片段
- 流程图

## 4. 交互关系
- 与其他模块的关系
- 数据流

## 5. 关键文件索引
| 文件 | 行数 | 说明 |
|------|------|------|

## 6. 待深入研究的问题

## 7. 总结
```

### 3.2 内容检查清单

- [ ] 标题清晰，描述性强
- [ ] 包含研究时间和状态
- [ ] 有核心概念解释
- [ ] 有关键代码位置和行号
- [ ] 有流程图或架构图 (如适用)
- [ ] 有关键文件索引表
- [ ] 有待研究问题列表
- [ ] 有总结要点

---

## 4. 具体整理任务

### 4.1 高优先级 (必须处理)

| 任务 | 描述 | 预计工作量 |
|------|------|-----------|
| T1 | 删除重复文件 `phase-prompt-system.md` 和 `prompt-system.md` 之一 | 5 分钟 |
| T2 | 确认 `conversation-storage-memory.md` 归属并重命名 | 15 分钟 |
| T3 | 创建 `README.md` 作为文档索引 | 30 分钟 |
| T4 | 为所有 Phase 报告添加标准头部 (研究时间、状态) | 1 小时 |

### 4.2 中优先级 (建议处理)

| 任务 | 描述 | 预计工作量 |
|------|------|-----------|
| T5 | 统一命名格式，添加 category 前缀 | 30 分钟 |
| T6 | 补充缺失的流程图/架构图 | 2-3 小时 |
| T7 | 统一关键文件索引表格式 | 1 小时 |

### 4.3 低优先级 (已完成)

| 任务 | 状态 | 说明 |
|------|------|------|
| T8 | ✅ 完成 | 已创建 03-glossary.md |
| T9 | ✅ 完成 | 已创建 04-faq.md |
| T10 | ⏸️ 跳过 | 保持扁平结构，README 索引已足够 |

---

## 5. 实施步骤 (已完成)

### 阶段 1: 清理和标准化 ✅

1. 删除重复文件 → `prompt-system.md` 已合并
2. 重命名文件 → 统一格式
3. 添加标准头部 → 15 个报告全部更新

### 阶段 2: 内容补充 ✅

1. 补充流程图 → 已在各报告中有架构图
2. 统一表格格式 → 关键文件索引统一命名
3. 补充待研究问题 → 各报告已有

### 阶段 3: 结构重组 ⏸️

1. ~~创建新目录结构~~ → 保持扁平结构
2. ~~移动文件~~ → 不需要
3. 创建 README 索引 → 已完成

### 阶段 4: 附录完善 ✅

1. 术语表 → 03-glossary.md
2. 常见问题 → 04-faq.md
3. 最终检查 → README.md 索引完整

---

## 6. 风险和注意事项

1. **文件删除风险**: 删除重复文件前需确认无重要差异
2. **链接失效**: 重组后需更新所有内部链接
3. **Git 历史**: 建议保留重命名文件的 Git 历史

---

## 7. 待用户确认

1. 是否同意上述目录结构？
2. 是否同意删除重复文件？
3. 是否需要保留 Phase 编号作为文件名前缀？
4. 文档整理的优先级是否正确？

---

## 8. 附件: 当前文件列表参考

```
.docs/
├── 01-deep-research-plan.md
└── reports/
    ├── phase-01-basic-architecture.md
    ├── phase-02-agent-loop.md
    ├── phase-03-provider.md
    ├── phase-04-skill.md
    ├── phase-05-mcp-integration.md
    ├── phase-06-acp-protocol.md
    ├── phase-07-config-system.md
    ├── phase-08-plugin-system.md
    ├── phase-09-auth-permission-env.md
    ├── phase-10-project-session-management.md
    ├── phase-11-infrastructure.md
    ├── phase-12-git-ide-integration.md
    ├── phase-13-sdk-extensions.md
    ├── phase-prompt-system.md        ⚠️ 重复
    ├── prompt-system.md             ⚠️ 重复
    └── conversation-storage-memory.md ⚠️ 未分类
```