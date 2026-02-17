# claude-assets 模块

[根目录](../CLAUDE.md) > **claude-assets**

---

> **Claude 配置资源仓库** - 包含 Claude Code 的所有配置资源，包括设置模板、自定义命令、智能体定义和输出风格

---

## 📋 目录

- [模块职责](#模块职责)
- [目录结构](#目录结构)
- [关键文件说明](#关键文件说明)
- [依赖关系](#依赖关系)
- [使用方法](#使用方法)
- [开发指南](#开发指南)
- [变更记录](#变更记录)

---

## 模块职责

`claude-assets/` 是 Claude Code 配置的**单一真实源（Single Source of Truth）**，负责：

1. **配置模板管理**：提供 `settings.yml.j2` Jinja2 模板，通过 Ansible 渲染为 `settings.json`
2. **自定义命令库**：存放 Markdown 格式的自定义命令定义（如 `/git-commit`、`/init-project`）
3. **智能体定义**：存放子 Agent 定义，用于复杂任务编排（如 `init-architect`）
4. **输出风格管理**：提供多种人格化输出风格（猫娘、大小姐、老王等工程师人设）
5. **全局指令文档**：维护 `CLAUDE.md`，定义 Claude 的全局行为规则

**核心原则**：

- 所有配置资源的**源文件**存放在此目录
- 通过 Ansible Playbook 同步到 `~/.claude/` 目录
- 支持版本控制与团队协作

---

## 目录结构

```text
claude-assets/
├── CLAUDE.md                # 全局指令文档（同步到 ~/.claude/CLAUDE.md）
├── settings.yml.j2          # settings.json 的 Jinja2 模板
├── agents/                  # 自定义智能体定义
│   └── mc/
│       ├── init-architect.md          # 项目初始化架构师智能体
│       └── get-current-datetime.md    # 获取当前时间智能体
├── commands/                # 自定义命令定义
│   └── mc/
│       ├── git-commit.md              # Git 提交命令
│       └── init-project.md            # 项目初始化命令
└── output-styles/           # 输出风格定义
    ├── engineer-professional.md       # 专业工程师风格
    ├── nekomata-engineer.md           # 猫娘工程师风格
    ├── ojousama-engineer.md           # 大小姐工程师风格
    └── laowang-engineer.md            # 老王工程师风格
```

---

## 关键文件说明

### 1. `settings.yml.j2`

**用途**：Claude Code 的核心配置模板，通过 Ansible 渲染为 JSON 格式。

**关键配置项**：

```yaml
# 环境变量（API 配置、模型选择）
env:
  ANTHROPIC_API_KEY: "{{ secrets.settings.env.ANTHROPIC_API_KEY }}"
  ANTHROPIC_BASE_URL: "{{ settings.env.ANTHROPIC_BASE_URL }}"
  ANTHROPIC_DEFAULT_OPUS_MODEL: >-
    {{ settings.env.ANTHROPIC_DEFAULT_OPUS_MODEL }}
  ANTHROPIC_DEFAULT_SONNET_MODEL: >-
    {{ settings.env.ANTHROPIC_DEFAULT_SONNET_MODEL }}
  ANTHROPIC_DEFAULT_HAIKU_MODEL: >-
    {{ settings.env.ANTHROPIC_DEFAULT_HAIKU_MODEL }}
  CLAUDE_CODE_SUBAGENT_MODEL: >-
    {{ settings.env.CLAUDE_CODE_SUBAGENT_MODEL }}

# 权限配置
permissions:
  allow: [Bash, Edit, Read, Write, WebSearch, ...]
  deny: []

# 输出风格
outputStyle: "{{ settings.outputStyle }}"

# 状态栏配置
statusLine:
  type: command
  command: ccline
```

**变量来源**：

- `secrets.settings.env.*`：来自 `inventory/default/group_vars/all/secrets.yml`（敏感信息）
- `settings.*`：来自 `inventory/default/group_vars/all/settings.yml`（公开配置）

**渲染流程**：

1. Ansible 读取变量
2. 使用 Jinja2 引擎渲染模板为 YAML
3. 转换为 JSON 格式输出到 `~/.claude/settings.json`

---

### 2. `CLAUDE.md`

**用途**：定义 Claude 的全局行为规则，所有 Claude Code 会话都会读取此文件。

**当前内容**：

```markdown
总是使用简体中文回复我。
```

**同步目标**：`~/.claude/CLAUDE.md`

---

### 3. `commands/` 目录

存放自定义命令定义，支持通过 `/command-name` 调用。

#### `commands/mc/git-commit.md`

**功能**：智能分析 Git 改动并生成 Conventional Commits 风格的提交信息。

**特性**：

- 支持自动暂存（`--all`）与拆分提交建议
- 支持 Emoji 提交信息（`--emoji`）
- 支持跳过 Git 钩子（`--no-verify`）
- 支持指定提交类型与作用域（`--type`、`--scope`）
- 根据 Git 历史自动选择提交信息语言（中文/英文）

**示例用法**：

```bash
/git-commit                           # 分析当前改动，生成提交信息
/git-commit --all                     # 暂存所有改动并提交
/git-commit --emoji                   # 使用 Emoji 提交信息
/git-commit --scope ui --type feat    # 指定作用域和类型
```

#### `commands/mc/init-project.md`

**功能**：初始化项目 AI 上下文，生成根级与模块级 CLAUDE.md 索引。

**特性**：

- 自动识别项目模块（monorepo 支持）
- 生成 Mermaid 结构图可视化架构
- 为每个模块添加导航面包屑
- 支持增量更新与断点续扫
- 输出覆盖率度量与下一步建议

**示例用法**：

```bash
/init-project my-claude-ansible 配置管理仓库
```

---

### 4. `agents/` 目录

存放子 Agent 定义，用于复杂任务的分解与编排。

#### `agents/mc/init-architect.md`

**功能**：自适应初始化项目架构文档，支持分阶段扫描。

**核心特性**：

- **阶段 A**：全仓清点（轻量级文件统计与模块发现）
- **阶段 B**：模块优先扫描（入口、接口、依赖、测试、数据模型）
- **阶段 C**：深度补捞（按需触发，填补缺失信息）
- 生成 Mermaid 结构图与面包屑导航
- 输出覆盖率报告与续扫建议

**工具权限**：`Read`, `Write`, `Glob`, `Grep`

#### `agents/mc/get-current-datetime.md`

**功能**：获取当前时间戳，用于文档生成时的时间标记。

---

### 5. `output-styles/` 目录

存放人格化输出风格定义，每个 `.md` 文件定义一种输出风格。

| 风格文件 | 人设特点 | 适用场景 |
| --------- | --------- | --------- |
| `engineer-professional.md` | 专业、简洁、标准 | 正式项目、技术文档 |
| `nekomata-engineer.md` | 可爱、活泼、猫娘 | 轻松氛围、个人项目 |
| `ojousama-engineer.md` | 优雅、礼貌、大小姐 | 注重礼节、高雅场景 |
| `laowang-engineer.md` | 幽默、接地气、老王 | 团队协作、日常开发 |

**切换方法**：

1. 编辑 `inventory/default/group_vars/all/settings.yml`
2. 修改 `settings.outputStyle` 为目标风格的文件名（不含 `.md`）
3. 运行 `ansible-playbook playbooks/sync_claude_config.yml`

---

## 依赖关系

### 上游依赖

- **Ansible 变量**：依赖 `inventory/default/group_vars/all/settings.yml` 和 `secrets.yml`
- **Jinja2 引擎**：用于渲染 `settings.yml.j2` 模板

### 下游依赖

- **Claude Code**：读取 `~/.claude/` 目录下的配置文件
- **Ansible Playbook**：通过 `playbooks/sync_claude_config.yml` 同步配置

### 模块依赖图

```text
inventory/settings.yml ──┐
inventory/secrets.yml  ──┤
                        ├──> Jinja2 渲染 settings.yml.j2 ──> ~/.claude/settings.json
                        │
                        └──> rsync 同步 commands/          ──> ~/.claude/commands/
                             rsync 同步 output-styles/     ──> ~/.claude/output-styles/
                             cp 同步 CLAUDE.md              ──> ~/.claude/CLAUDE.md
```

---

## 使用方法

### 1. 修改配置模板

编辑 `settings.yml.j2` 添加新的配置项：

```yaml
# 添加新的环境变量
env:
  MY_CUSTOM_VAR: "{{ settings.myCustomVar }}"
```

### 2. 添加自定义命令

1. 在 `commands/mc/` 创建新文件 `my-command.md`
2. 编写 front-matter 和指令内容：

```markdown
---
description: 我的自定义命令
allowed-tools: Read(**), Write(**)
argument-hint: [选项]
---

# My Custom Command

这里是命令的详细说明...
```

1. 同步配置：`ansible-playbook playbooks/sync_claude_config.yml`
2. 使用命令：`/my-command`

### 3. 添加新的输出风格

1. 在 `output-styles/` 创建新文件 `my-style.md`
2. 编写人格化指令（参考现有风格）
3. 同步配置：`ansible-playbook playbooks/sync_claude_config.yml`
4. 在 `settings.yml` 中设置 `outputStyle: "my-style"`

### 4. 修改全局指令

编辑 `CLAUDE.md` 添加新的全局规则：

```markdown
总是使用简体中文回复我。
代码注释也使用中文。
```

同步后所有 Claude 会话都会遵循这些规则。

---

## 开发指南

### 命令开发规范

**Front-matter 必需字段**：

```yaml
description: 命令的简短描述（一句话）
allowed-tools: 允许使用的工具列表（如 Read(**), Bash(**)）
argument-hint: 参数提示（如 [--option] <required>）
```

**可选字段**：

```yaml
confirm: true          # 是否需要用户确认
examples:             # 使用示例列表
  - /my-command --help
  - /my-command input.txt
```

**正文编写原则**：

1. **清晰的用法说明**：提供多个示例
2. **详细的选项说明**：解释每个参数的作用
3. **工作流程描述**：说明命令的执行步骤
4. **最佳实践建议**：给出使用建议

### 智能体开发规范

**Front-matter 必需字段**：

```yaml
name: agent-name              # 智能体名称
description: 智能体描述
tools: Read, Write, Glob      # 允许的工具列表
color: orange                 # UI 颜色（可选）
```

**正文编写原则**：

1. **明确的职责定义**：说明智能体的目标
2. **详细的执行策略**：分阶段描述工作流程
3. **约束与边界**：明确不能做什么
4. **输出规范**：定义返回格式

### 输出风格开发规范

**结构建议**：

```markdown
# 人格设定

你是一个[人设描述]...

## 语言风格

- 使用[特定词汇]
- 语气[特点]
- 口头禅：[示例]

## 技术输出规范

- 代码块：[格式要求]
- 错误提示：[风格要求]
- 解释方式：[要求]
```

---

## 变更记录

### 2026-02-17

- **feat(docs)**: 初始化 claude-assets 模块文档（CLAUDE.md）
- **feat(docs)**: 新增面包屑导航链接到根目录

### 2026-02-12

- **feat(commands)**: 新增 git-commit 自定义命令
- **feat(commands)**: 新增 init-project 自定义命令

### 2026-02-11

- **refactor(config)**: 细化 settings.yml.j2 中的模型配置结构

### 2026-02-10

- **feat(agents)**: 新增 init-architect 智能体定义
- **feat(agents)**: 新增 get-current-datetime 智能体定义

---

**最后更新时间**: 2026-02-17T05:32:00+00:00
