# agents 模块

[根目录](../../CLAUDE.md) > [claude-assets](../CLAUDE.md) > **agents**

---

> **自定义智能体库** - 存放 Claude Code 的子 Agent 定义，用于复杂任务的分解与编排

---

## 📋 目录

- [模块职责](#模块职责)
- [目录结构](#目录结构)
- [关键文件说明](#关键文件说明)
- [使用方法](#使用方法)
- [开发指南](#开发指南)
- [常见问题](#常见问题)

---

## 模块职责

`agents/` 目录负责管理 Claude Code 的子智能体定义，提供：

1. **任务编排机制**：将复杂任务分解为多个子任务，由不同的智能体协作完成
2. **智能体规范文档**：front-matter 定义智能体元数据（名称、描述、工具权限、UI 配色）
3. **执行策略指引**：正文提供详细的工作流程、约束条件和输出规范
4. **智能体分组管理**：通过子目录（如 `mc/`）组织不同类别的智能体

**核心概念**：

- **主 Agent**：Claude Code 主会话中的 AI Agent
- **子 Agent（Sub-agent）**：由主 Agent 调用的专用 Agent，负责特定子任务
- **编排（Orchestration）**：主 Agent 通过命令调用子 Agent，并传递参数与上下文

---

## 目录结构

```text
agents/
└── mc/                              # mc 智能体组（项目管理相关）
    ├── init-architect.md            # 项目初始化架构师智能体
    └── get-current-datetime.md      # 获取当前时间智能体
```

**命名规范**：

- 智能体文件名使用 `kebab-case.md` 格式（如 `init-architect.md`）
- 调用时使用智能体名称：`init-architect`（不需要斜杠前缀）
- 支持子目录分组：`mc/init-architect`（可选路径前缀）

---

## 关键文件说明

### 1. `mc/init-architect.md`

**用途**：自适应初始化项目架构文档，支持分阶段扫描与增量更新。

**核心功能**：

- **全仓清点**：使用 `Glob` 分批获取文件清单，识别模块候选（`package.json`、`pyproject.toml`、`go.mod` 等）
- **模块扫描**：对每个模块按优先级读取（入口、接口、依赖、数据模型、测试、配置）
- **深度补捞**：按需触发深度扫描，填补缺失信息
- **文档生成**：生成根级与模块级 `CLAUDE.md`，包含 Mermaid 结构图与面包屑导航
- **覆盖率报告**：输出扫描覆盖率、忽略统计与续扫建议

**工具权限**：

- `Read`：读取项目文件
- `Write`：写入 `CLAUDE.md` 和 `.claude/index.json`
- `Glob`：文件模式匹配
- `Grep`：内容搜索

**输入参数**（通过命令传递）：

- `project_summary`：项目摘要或名称
- `current_timestamp`：当前时间戳（ISO-8601 格式）

**执行策略**（自动选择强度）：

1. **阶段 A：全仓清点（轻量）**
   - 使用多次 `Glob` 分批获取文件清单（避免单次超限）
   - 统计文件计数、语言占比、目录拓扑
   - 识别模块候选（`package.json`、`pyproject.toml`、`go.mod`、
     `Cargo.toml`、`apps/*`、`packages/*` 等）
   - 为每个候选模块标注：语言、入口文件猜测、测试目录是否存在、
     配置文件是否存在

2. **阶段 B：模块优先扫描（中等）**
   - 对每个模块按以下顺序尝试读取（分批、分页）：
     - **入口与启动**：`main.ts`/`index.ts`/`cmd/*/main.go`/`app.py`/`src/main.rs` 等
     - **对外接口**：路由、控制器、API 定义、proto/openapi
     - **依赖与脚本**：`package.json scripts`、`pyproject.toml`、`go.mod`、`Cargo.toml`、配置目录
     - **数据层**：`schema.sql`、`prisma/schema.prisma`、ORM 模型、迁移目录
     - **测试**：`tests/**`、`__tests__/**`、`*_test.go`、`*.spec.ts` 等
     - **质量工具**：`eslint/ruff/golangci` 等配置
   - 形成"模块快照"，只抽取高信号片段与路径，不粘贴大段代码

3. **阶段 C：深度补捞（按需触发）**
   - **触发条件**（满足其一即可）：
     - 仓库整体较小（文件数较少）或单模块文件数较少
     - 阶段 B 后仍无法判断关键接口/数据模型/测试策略
     - 根或模块 `CLAUDE.md` 缺信息项
   - **动作**：对目标目录追加分页读取，补齐缺项

**产物与增量更新**：

1. **根级 `CLAUDE.md`**
   - 如果已存在，则在顶部插入/更新 `变更记录 (Changelog)`
   - 根级结构（精简而全局）：
     - 项目愿景
     - 架构总览
     - **模块结构图（Mermaid）**：在"模块索引"表格上方生成 `graph TD` 树形图，每个节点可点击并链接到对应模块的 `CLAUDE.md`
     - 模块索引（表格形式）
     - 运行与开发
     - 测试策略
     - 编码规范
     - AI 使用指引
     - 变更记录 (Changelog)

2. **模块级 `CLAUDE.md`**
   - 放在每个模块目录下，结构建议：
     - **相对路径面包屑**：在每个模块 `CLAUDE.md` 的最顶部，
       插入一行相对路径面包屑，链接到各级父目录及根 `CLAUDE.md`
       - 示例（位于 `packages/auth/CLAUDE.md`）：
         `[根目录](../../CLAUDE.md) > [packages](../) > **auth**`
     - 模块职责
     - 入口与启动
     - 对外接口
     - 关键依赖与配置
     - 数据模型
     - 测试与质量
     - 常见问题 (FAQ)
     - 相关文件清单
     - 变更记录 (Changelog)

3. **`.claude/index.json`**
   - 记录：当前时间戳、根/模块列表、每个模块的入口/接口/测试/重要路径、扫描覆盖率、忽略统计、是否因上限被截断（`truncated: true`）

**覆盖率与可续跑**：

- 每次运行都计算并打印：
  - 估算总文件数、已扫描文件数、覆盖百分比
  - 每个模块的覆盖摘要与缺口（缺接口、缺测试、缺数据模型等）
  - 被忽略/跳过的 Top 目录与原因（忽略规则/大文件/时间或调用上限）
- 将"缺口清单"写入 `index.json`，下次运行时优先补齐缺口（断点续扫）

**结果摘要（打印到主对话）**：

- 根/模块 `CLAUDE.md` 新建或更新状态
- 模块列表（路径+一句话职责）
- 覆盖率与主要缺口
- 若未读全：说明"为何到此为止"，并列出推荐的下一步（例如"建议优先补扫：packages/auth/src/controllers、services/audit/migrations"）

---

### 2. `mc/get-current-datetime.md`

**用途**：获取当前时间戳，用于文档生成时的时间标记。

**核心功能**：

- 返回 ISO-8601 格式的当前时间戳（如 `2026-02-17T05:32:00+00:00`）
- 确保所有文档的时间标记一致且准确

**工具权限**：

- 无（仅计算当前时间）

**输出格式**：

```json
{
  "timestamp": "2026-02-17T05:32:00+00:00"
}
```

---

## 使用方法

### 调用智能体

智能体通常由自定义命令调用，不直接在 CLI 中使用。

**示例**（在 `init-project` 命令中）：

```markdown
## 编排说明

**步骤 1**：调用 `get-current-datetime` 子智能体获取当前时间戳。

**步骤 2**：调用一次 `init-architect` 子智能体，输入：

- `project_summary`: $ARGUMENTS
- `current_timestamp`: (来自步骤1的时间戳)
```

### 查看智能体文档

查看智能体的 front-matter 和正文：

```bash
cat ~/.claude/agents/mc/init-architect.md
```

### 验证智能体同步

检查智能体是否已同步到 `~/.claude/agents/`：

```bash
ls -la ~/.claude/agents/mc/
```

---

## 开发指南

### 智能体文件结构

**标准模板**：

```markdown
---
name: agent-name
description: 智能体的简短描述
tools: Read, Write, Glob, Grep
color: orange
---

# 智能体名称

> 智能体的详细说明

## 一、通用约束

- 约束 1
- 约束 2

## 二、执行策略

1. 步骤 1
2. 步骤 2

## 三、产物与输出

- 输出 1
- 输出 2

## 四、结果摘要

- 摘要内容
```

---

### Front-matter 字段说明

**必需字段**：

- `name`：智能体名称（用于调用）
- `description`：智能体的简短描述（显示在智能体列表中）
- `tools`：允许使用的工具列表（逗号分隔）
  - `Read`, `Write`, `Edit`：文件操作
  - `Glob`, `Grep`：文件搜索与内容搜索
  - `Bash`：执行 Bash 命令
  - `WebFetch`, `WebSearch`：网络请求

**可选字段**：

- `color`：UI 配色（如 `orange`, `blue`, `green`）
- `confirm: true`：执行前需要用户确认（危险操作建议开启）

---

### 正文编写指南

**必需章节**：

1. **通用约束**：明确智能体的边界条件（不能做什么）
2. **执行策略**：详细的工作流程（分阶段、分步骤说明）
3. **产物与输出**：定义智能体的输出格式与文件清单

**可选章节**：

1. **覆盖率与可续跑**：说明如何计算覆盖率与支持增量更新
2. **结果摘要**：定义打印到主对话的摘要格式

**编写原则**：

- **清晰性**：每个步骤都应有明确的输入输出说明
- **完整性**：覆盖常见使用场景和边界情况
- **安全性**：明确说明智能体的副作用和危险操作
- **可测试性**：提供足够的约束，确保智能体行为可预测

---

### 添加新智能体

**步骤 1**：创建智能体文件

```bash
# 在 agents/mc/ 创建新文件
touch claude-assets/agents/mc/my-agent.md
```

**步骤 2**：编写智能体定义

```markdown
---
name: my-agent
description: 我的自定义智能体
tools: Read, Write
color: blue
---

# 我的智能体

> 详细说明...

## 一、通用约束

- 只读/写文档，不改源代码

## 二、执行策略

1. 步骤 1
2. 步骤 2

## 三、产物与输出

- 输出文件 1
- 输出文件 2
```

**步骤 3**：同步配置

```bash
ansible-playbook playbooks/sync_claude_config.yml
```

**步骤 4**：在命令中调用

编辑 `commands/mc/my-command.md`，添加智能体调用：

```markdown
## 编排说明

调用 `my-agent` 子智能体，输入：

- `param1`: 值 1
- `param2`: 值 2
```

---

### 工具权限管理

**工具列表**：

- `Read`：读取文件
- `Write`：写入文件
- `Edit`：编辑文件
- `Glob`：文件模式匹配
- `Grep`：内容搜索
- `Bash`：执行 Bash 命令（慎用）
- `WebFetch`, `WebSearch`：网络请求

**权限原则**：

- **最小权限**：只授予必需的工具权限
- **明确边界**：在"通用约束"章节明确不能做什么
- **避免 Bash**：尽量使用专用工具（`Read`/`Write`/`Glob`/`Grep`），避免使用 `Bash`

**示例**：

```yaml
# 推荐（具体）
tools: Read, Write, Glob, Grep

# 不推荐（过于宽泛）
tools: Bash
```

---

## 常见问题

### Q1：智能体未生效？

**原因**：可能未同步到 `~/.claude/agents/`

**解决方法**：

```bash
# 重新同步配置
ansible-playbook playbooks/sync_claude_config.yml

# 检查文件是否存在
ls ~/.claude/agents/mc/
```

---

### Q2：智能体权限不足？

**原因**：`tools` 配置过于严格

**解决方法**：

编辑智能体文件的 front-matter，添加所需工具：

```yaml
tools: Read, Write, Glob, Grep
```

---

### Q3：如何调试智能体？

**方法 1**：查看智能体文档

```bash
cat ~/.claude/agents/mc/my-agent.md
```

**方法 2**：在命令中打印智能体输出

在命令的"编排说明"中添加日志输出要求。

---

### Q4：智能体命名冲突？

**原因**：不同智能体组使用了相同的智能体名

**解决方法**：

- 使用完整路径调用：`mc/init-architect`
- 或重命名智能体文件：`init-architect-mc.md`

---

**最后更新时间**: 2026-02-17T05:32:00+00:00
