# commands 模块

[根目录](../../CLAUDE.md) > [claude-assets](../CLAUDE.md) > **commands**

---

> **自定义命令库** - 存放 Claude Code 的自定义命令定义（Markdown 格式），通过 `/command-name` 调用

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

`commands/` 目录负责管理 Claude Code 的自定义命令定义，提供：

1. **命令扩展机制**：通过 Markdown 格式定义新命令，无需修改 Claude Code 源码
2. **命令规范文档**：front-matter 定义命令元数据（描述、工具权限、参数提示）
3. **命令实现指引**：正文提供详细的执行逻辑、最佳实践和示例
4. **命令分组管理**：通过子目录（如 `mc/`）组织不同类别的命令

---

## 目录结构

```text
commands/
└── mc/                          # mc 命令组（项目管理相关）
    ├── git-commit.md            # Git 提交命令
    └── init-project.md          # 项目初始化命令
```

**命名规范**：

- 命令文件名使用 `kebab-case.md` 格式（如 `git-commit.md`）
- 调用时使用斜杠前缀：`/git-commit`
- 支持子目录分组：`/mc/git-commit`（可选路径前缀）

---

## 关键文件说明

### 1. `mc/git-commit.md`

**用途**：智能分析 Git 改动并生成 Conventional Commits 风格的提交信息。

**核心功能**：

- **改动分析**：读取 `git status` 和 `git diff` 输出，识别已暂存和未暂存的改动
- **提交类型推断**：根据改动内容自动判断 `feat`/`fix`/`docs`/`refactor`/`chore` 等类型
- **拆分建议**：检测多组独立变更，建议拆分为多次提交（原子提交原则）
- **消息生成**：生成符合 Conventional Commits 规范的提交信息（支持 Emoji）
- **语言自适应**：根据 Git 历史提交的主要语言选择中文/英文
- **钩子支持**：默认运行本地 Git 钩子，支持 `--no-verify` 跳过

**参数说明**：

```bash
/git-commit                           # 分析当前改动，生成提交信息
/git-commit --all                     # 暂存所有改动并提交
/git-commit --no-verify               # 跳过 Git 钩子检查
/git-commit --emoji                   # 使用 Emoji 提交信息
/git-commit --scope ui --type feat    # 指定作用域和类型
/git-commit --amend --signoff         # 修补上次提交并签名
```

**工作流程**：

1. 仓库/分支校验（检查 Git 状态、冲突等）
2. 改动检测（`git status --porcelain`, `git diff`）
3. 拆分建议（按关注点、文件模式、改动类型聚类）
4. 提交信息生成（推断 type/scope，生成主题与正文）
5. 执行提交（`git commit -F .git/COMMIT_EDITMSG`）
6. 安全回滚（提供 `git restore --staged` 指令）

**Type 与 Emoji 映射**（使用 `--emoji` 时）：

- ✨ `feat`：新增功能
- 🐛 `fix`：缺陷修复
- 📝 `docs`：文档与注释
- 🎨 `style`：风格/格式
- ♻️ `refactor`：重构
- ⚡️ `perf`：性能优化
- ✅ `test`：测试
- 🔧 `chore`：构建/工具/杂务
- 👷 `ci`：CI/CD 配置
- ⏪️ `revert`：回滚提交

---

### 2. `mc/init-project.md`

**用途**：初始化项目 AI 上下文，生成根级与模块级 CLAUDE.md 索引。

**核心功能**：

- **根级文档生成**：创建/更新根目录 `CLAUDE.md`（项目愿景、架构总览、模块索引）
- **模块级文档生成**：为每个识别的模块生成本地 `CLAUDE.md`（职责、入口、依赖、测试）
- **Mermaid 结构图**：自动生成可视化的模块依赖图（可点击链接）
- **导航面包屑**：为每个模块文档添加相对路径面包屑（如 `[根目录](../../CLAUDE.md) > [packages](../) > **auth**`）
- **增量更新**：支持断点续扫，输出覆盖率度量与下一步建议

**参数说明**：

```bash
/init-project <项目摘要或名称>
```

**示例**：

```bash
/init-project my-claude-ansible 配置管理仓库
```

**编排说明**：

该命令通过子智能体编排实现：

1. 调用 `get-current-datetime` 智能体获取当前时间戳
2. 调用 `init-architect` 智能体执行初始化（传入项目摘要与时间戳）

**执行策略**（由 Agent 自适应决定）：

- **阶段 A**：全仓清点（轻量级文件统计与模块发现）
- **阶段 B**：模块优先扫描（入口、接口、依赖、测试、数据模型）
- **阶段 C**：深度补捞（按需触发，填补缺失信息）

**输出要求**：

- 打印初始化结果摘要（创建/更新的文档、识别的模块数量）
- 明确提及"已生成 Mermaid 结构图"和"已为 N 个模块添加导航面包屑"
- 输出覆盖率度量与主要缺口
- 若未读全：列出推荐的下一步扫描路径

---

## 使用方法

### 调用命令

在 Claude Code CLI 中直接输入命令：

```bash
# 方式 1：直接调用（推荐）
/git-commit --emoji

# 方式 2：带路径前缀调用
/mc/git-commit --emoji
```

### 查看命令帮助

查看命令的 front-matter 和正文：

```bash
cat ~/.claude/commands/mc/git-commit.md
```

### 验证命令同步

检查命令是否已同步到 `~/.claude/commands/`：

```bash
ls -la ~/.claude/commands/mc/
```

---

## 开发指南

### 命令文件结构

**标准模板**：

```markdown
---
description: 命令的简短描述（一句话）
allowed-tools: Read(**), Exec(git), Write(.git/COMMIT_EDITMSG)
argument-hint: [--option] <required-arg>
# examples:
#   - /my-command --help
#   - /my-command input.txt
---

# Claude Command: My Command

命令的详细说明...

## Usage

\`\`\`bash
/my-command [options] <args>
\`\`\`

### Options

- `--option`：选项说明

## What This Command Does

1. 步骤 1
2. 步骤 2
3. 步骤 3

## Best Practices

- 最佳实践 1
- 最佳实践 2

## Examples

\`\`\`bash
/my-command --option value
\`\`\`
```

---

### Front-matter 字段说明

**必需字段**：

- `description`：命令的简短描述（显示在命令列表中）
- `allowed-tools`：允许使用的工具列表（限制命令的权限）
  - `Read(**)`：读取任意文件
  - `Write(path/to/file)`：写入特定文件
  - `Exec(git, npm)`：执行特定命令
  - `Bash(**)`：执行任意 Bash 命令
- `argument-hint`：参数提示（显示在命令自动完成中）

**可选字段**：

- `confirm: true`：执行前需要用户确认（危险操作建议开启）
- `examples: [...]`：使用示例列表（多行 YAML 数组）

---

### 正文编写指南

**必需章节**：

1. **Usage**：命令的基本用法（命令格式、参数说明）
2. **What This Command Does**：详细的工作流程（分步骤说明）
3. **Examples**：实际使用示例（多个场景）

**可选章节**：

1. **Best Practices**：最佳实践建议
2. **Important Notes**：重要注意事项
3. **Guidelines**：使用指南

**编写原则**：

- **清晰性**：每个步骤都应有明确的输入输出说明
- **完整性**：覆盖常见使用场景和边界情况
- **安全性**：明确说明命令的副作用和危险操作
- **示例驱动**：提供丰富的使用示例和对比

---

### 添加新命令

**步骤 1**：创建命令文件

```bash
# 在 commands/mc/ 创建新文件
touch claude-assets/commands/mc/my-command.md
```

**步骤 2**：编写命令定义

```markdown
---
description: 我的自定义命令
allowed-tools: Read(**), Write(**)
argument-hint: [--option] <input>
---

# Claude Command: My Command

这里是命令的详细说明...
```

**步骤 3**：同步配置

```bash
ansible-playbook playbooks/sync_claude_config.yml
```

**步骤 4**：验证命令

```bash
# 检查文件是否同步
ls ~/.claude/commands/mc/my-command.md

# 在 Claude Code 中调用
/my-command --help
```

---

### 工具权限管理

**工具列表**：

- `Read(path/pattern)`：读取文件（支持通配符）
- `Write(path/pattern)`：写入文件
- `Edit(path/pattern)`：编辑文件
- `Exec(command1, command2)`：执行特定命令
- `Bash(**)`：执行任意 Bash 命令（慎用）
- `Glob`, `Grep`：文件搜索与内容搜索
- `WebFetch`, `WebSearch`：网络请求

**权限原则**：

- **最小权限**：只授予必需的工具权限
- **路径限制**：尽量指定具体的文件路径或模式
- **命令限制**：`Exec` 应列出具体的命令名称，避免使用 `Bash(**)`

**示例**：

```yaml
# 推荐（具体）
allowed-tools: Read(src/**/*.js, package.json), Exec(git, npm test)

# 不推荐（过于宽泛）
allowed-tools: Read(**), Bash(**)
```

---

## 常见问题

### Q1：命令未生效？

**原因**：可能未同步到 `~/.claude/commands/`

**解决方法**：

```bash
# 重新同步配置
ansible-playbook playbooks/sync_claude_config.yml

# 检查文件是否存在
ls ~/.claude/commands/mc/
```

---

### Q2：命令权限不足？

**原因**：`allowed-tools` 配置过于严格

**解决方法**：

编辑命令文件的 front-matter，添加所需工具：

```yaml
allowed-tools: Read(**), Write(**), Exec(git)
```

---

### Q3：如何调试命令？

**方法 1**：查看命令文档

```bash
cat ~/.claude/commands/mc/my-command.md
```

**方法 2**：启用详细日志

在 Claude Code 中查看执行日志（如果支持）。

---

### Q4：命令命名冲突？

**原因**：不同命令组使用了相同的命令名

**解决方法**：

- 使用完整路径调用：`/mc/git-commit`
- 或重命名命令文件：`git-commit-mc.md`

---

**最后更新时间**: 2026-02-17T05:32:00+00:00
