# my-claude-ansible

> **Claude Code 配置管理仓库** - 基于 Ansible 的声明式配置管理，实现 Claude 个性化配置的自动化部署与同步

---

## 开发约定（AI 助手必读）

### my-claude CLI 工具

- **定位**：基于 Typer 的 Python CLI，是 Ansible 配置管理的用户友好前端
- **源码**：`my_claude/` 包，入口 `my_claude/cli.py`，pyproject.toml 注册为 `my-claude` 命令
- **安装**：`uv tool install -e .`（全局）或 `uv pip install -e .`（开发）
- **核心命令**：`my-claude status`（查看状态）、`my-claude provider use <name>`（切换提供商）、`my-claude sync`（同步配置）
- **工作流**：CLI 修改 `inventory/` 下的 YAML → 可选 `--sync` 自动调用 `task sync` → Ansible 渲染并部署到 `~/.claude/`
- **技术栈**：Typer（CLI 框架）、Rich（终端 UI）、ruamel.yaml（保留注释的 YAML 读写）
- **Typer 注意**：所有 `typer.Typer()` 须传 `context_settings={"help_option_names": ["-h", "--help"]}` 以支持 `-h` 短选项

- **Python 命令前缀**：所有 Python/Ansible 命令必须使用 `uv run` 前缀（如 `uv run ansible-playbook`）
- **任务管理**：使用 `Taskfile.yml` 统一管理所有任务，运行 `task --list` 查看可用任务
- **常用任务**：
  - `task deploy` - 完整部署
  - `task sync` - 仅同步配置
  - `task sync-claude-json` - 仅同步 ~/.claude.json（deep merge 模式）
  - `task check-claude-json` - 预览 claude.json 变更
  - `task check` - 预览完整变更
- **配置同步**：更新 settings.yml 后使用 `task sync` 同步到 `~/.claude/`
- **配置备份**：同步时自动备份旧配置到 `tmps/backup/` 目录，格式：`settings.json.YYYYMMDD_HHMMSS`、`claude.json.YYYYMMDD_HHMMSS`
- **确认机制**：`confirm_settings_update` 控制 settings.json 更新，`claude_json.confirm_claude_json_update` 控制 claude.json 更新；使用 `skip_confirm=true` 可跳过所有确认（CI/CD 用）
- **预览模式**：`task check` / `task check-sync` 使用 Ansible 的 `--check --diff` 模式
- **Playbook 语法检查**：使用 `uv run ansible-playbook --syntax-check` 验证 Playbook 语法
- **模板渲染测试**：使用 `uv run ansible -m template` 快速验证单模板渲染
- **深度合并语法**：使用 `jq -s '.[0] * .[1]'` 递归合并 JSON 文件
- **配置搜索**：使用 `grep -r --include="*.yml" --include="*.j2" --include="*.yaml"` 搜索项目配置

### 注意事项

- **--check 模式限制**：`--check` 模式下 template 模块不会创建文件，依赖文件的任务可能失败
- **变量覆盖风险**：使用 `-e '{"key": {"subkey": "value"}}'` 格式会**完全覆盖**原变量
- **正确的变量传递**：使用顶层变量（如 `skip_confirm=true`）避免嵌套变量覆盖问题
- **Skills 目录结构**：自定义 skills 位于 `claude-assets/skills/`，每个 skill 在独立子目录中，文件名为 `SKILL.md`
- **Skill 命名规范**：front-matter 必须包含 `name:` 字段，部分 skills 使用 `ms-` 前缀（如 ms-git-commit）
- **项目级 Skills**：项目根目录的 `.claude/skills/` 用于存放仅在当前项目中使用的 skills，不会被同步到 `~/.claude/`
- **.gitignore 注意**：仅忽略 `.claude/settings.local.json`，不要忽略整个 `.claude/` 目录（项目级 skills 存放在这里）

### ⚠️ MCP 配置重要提示

- **不要**在 `settings.yml.j2` 或 `settings.json` 中添加 `mcpServers` 配置项（不在官方 schema 中）
- 使用 `claude_json.mcpServers` 变量（在 `inventory/default/group_vars/all/mcp_servers.yml` 中定义）管理自定义 MCP 服务器
- 配置通过 `claude-assets/claude.yml.j2` 模板渲染后与 `~/.claude.json` 做递归深度合并（`jq deep merge`）

---

## 📋 目录

- [项目愿景](#项目愿景)
- [架构总览](#架构总览)
- [模块索引](#模块索引)
- [快速开始](#快速开始)
- [运行与开发](#运行与开发)
- [全局开发规范](#全局开发规范)
- [AI 使用指引](#ai-使用指引)
- [CI/CD 集成](#cicd-集成)

---

## 项目愿景

**my-claude-ansible** 旨在提供一个可重复、可版本控制的 Claude Code 配置管理解决方案，通过 Ansible 的基础设施即代码（IaC）理念，实现：

- **声明式配置管理**：使用 YAML 声明期望状态，由 Ansible 自动执行到位
- **个性化输出风格**：支持多种人格化输出风格（猫娘工程师、大小姐工程师、老王工程师等）
- **自定义命令扩展**：通过 Markdown 定义 Claude 自定义命令（如 git-commit、init-project）
- **模型配置灵活性**：支持多层级模型配置（Opus、Sonnet、Haiku、子代理模型）
- **跨环境一致性**：确保开发、测试、生产环境的 Claude 配置一致

---

## 架构总览

本项目采用标准的 Ansible 项目结构，核心组件包括：

```text
my-claude/
├── claude-assets/        # Claude 配置资源（源文件）
│   ├── agents/          # 自定义智能体定义
│   ├── skills/        # 自定义命令定义
│   ├── output-styles/   # 输出风格定义
│   ├── CLAUDE.md        # 全局指令文档
│   └── settings.yml.j2  # settings.json 的 Jinja2 模板
├── inventory/           # Ansible 清单与变量
│   └── default/
│       ├── inventory.yml         # 主机清单
│       └── group_vars/all/
│           ├── settings.yml      # 公开配置变量
│           └── secrets.yml       # 敏感配置（API Key 等）
├── playbooks/           # Ansible Playbook
│   └── setup.yml                # 一键部署（安装插件 + 同步配置）
├── ansible.cfg          # Ansible 全局配置
└── tmps/                # 临时文件与日志
    ├── ansible.log      # Ansible 执行日志
    └── facts/           # Facts 缓存目录
```

**工作流程**：

1. 在 `inventory/default/group_vars/all/settings.yml` 中声明配置变量
2. Ansible 读取变量并渲染 `claude-assets/settings.yml.j2` 模板
3. 将渲染结果转换为 JSON 格式输出到 `~/.claude/settings.json`
4. 使用 rsync 同步 `skills`、`output-styles`、`CLAUDE.md` 等资源文件到 `~/.claude/`

---

## 模块索引

| 模块路径                       | 职责                                                      | 关键文件                                                 |
| ------------------------------ | --------------------------------------------------------- | -------------------------------------------------------- |
| `my_claude/`                   | Python CLI 工具，Ansible 配置管理的命令行前端             | `cli.py`, `config.py`, `models.py`, `sync.py`            |
| `claude-assets/`               | Claude 配置资源仓库，包含模板、命令、智能体、输出风格定义 | `settings.yml.j2`, `CLAUDE.md`                           |
| `claude-assets/skills/`        | 自定义命令定义（Markdown 格式）                           | `git-commit.md`, `git-sync-branch.md`, `init-project.md` |
| `claude-assets/agents/`        | 自定义智能体定义（子 Agent）                              | `init-architect.md`, `get-current-datetime.md`           |
| `claude-assets/output-styles/` | 个性化输出风格定义（人格化）                              | `nekomata-engineer.md`, `laowang-engineer.md` 等         |
| `inventory/`                   | Ansible 清单与变量管理                                    | `inventory.yml`, `settings.yml`, `secrets.yml`           |
| `playbooks/`                   | Ansible Playbook 剧本                                     | `setup.yml`（通过 tags 控制执行阶段）                    |

---

## 快速开始

### 前置条件

- Claude CLI（需用户预先安装；未安装时 `setup.yml` 会提示并停止）
- Python 3.8+
- Ansible 2.9+（推荐通过 uv 管理）
- Node.js 18+（用于安装 ccline）

**命令执行说明**：

- 本项目使用 uv 管理 Python 依赖，所有 `ansible-playbook` 命令需添加 `uv run` 前缀
- 如果你已全局安装 Ansible，可直接使用 `ansible-playbook` 命令（不加前缀）

### 一键部署（推荐）

```bash
# 1. 克隆仓库
git clone <仓库地址> my-claude
cd my-claude

# 2. 安装 Claude CLI（需用户自行完成）
curl -fsSL https://claude.ai/install.sh | bash

# 3. 配置变量
# 编辑 inventory/default/group_vars/all/settings.yml（公开配置）
vim inventory/default/group_vars/all/settings.yml
# 编辑 inventory/default/group_vars/all/secrets.yml（敏感信息）
vim inventory/default/group_vars/all/secrets.yml

# 4. 一键部署（检查 Claude CLI + 安装插件 + 同步配置）
uv run ansible-playbook playbooks/setup.yml
```

**`setup.yml` 会自动执行**：

1. 检查 Claude CLI 是否已安装（未安装会提示并停止，不自动安装）
2. 自动安装 `enabled_plugins` 列表中的插件
3. 同步配置到 `~/.claude/settings.json`
4. 验证部署结果

### 常用选项

```bash
# 跳过插件安装（仅同步配置）
uv run ansible-playbook playbooks/setup.yml --tags sync_config

# 仅安装插件
uv run ansible-playbook playbooks/setup.yml --tags install_plugins

# 查看配置变量（不执行）
uv run ansible-playbook playbooks/setup.yml --check --diff
```

### 验证配置

```bash
# 检查 settings.json 是否正确生成
cat ~/.claude/settings.json | jq .

# 检查自定义命令是否同步
ls -la ~/.claude/skills/

# 检查输出风格是否同步
ls -la ~/.claude/output-styles/
```

---

## 运行与开发

### 主要命令（推荐使用 Taskfile）

**安装 Taskfile**：

```bash
# 方式 1: 使用 Go 安装
go install github.com/go-task/task/v3/cmd/task@latest

# 方式 2: 使用 Homebrew
brew install go-task

# 方式 3: 其他安装方式见 https://taskfile.dev/installation/
```

**使用 Taskfile**：

```bash
# 查看所有可用任务
task --list

# 一键部署（推荐）
task deploy

# 同步配置到 ~/.claude（常用）
task sync

# 仅安装插件
task install-plugins

# 预览配置变更
task check-sync

# 完整预览变更
task check

# 查看当前配置
task view-settings

# 查看所有可用 tags
task list-tags
```

**Taskfile 任务列表**：

| 任务                     | 说明                              |
| ------------------------ | --------------------------------- |
| `task deploy`            | 完整部署（安装插件 + 同步配置）   |
| `task sync`              | 仅同步配置                        |
| `task install-plugins`   | 仅安装插件                        |
| `task sync-claude-json`  | 仅同步 claude.json（deep merge）  |
| `task check-claude-json` | 预览 claude.json 变更（不执行）   |
| `task check`             | 预览完整部署变更（不执行）        |
| `task check-sync`        | 预览配置同步变更（不执行）        |
| `task sync-force`        | 同步配置，跳过确认（CI/CD 用）    |
| `task deploy-force`      | 完整部署，跳过确认（CI/CD 用）    |
| `task list-tags`         | 查看所有可用 tags                 |
| `task list-tasks`        | 查看所有 Ansible 任务             |
| `task view-settings`     | 查看当前 ~/.claude/settings.json  |
| `task view-claude-json`  | 查看当前 ~/.claude.json（格式化） |

**说明**：

- 日常使用推荐 `Taskfile` 方式
- CI/CD 环境使用带 `-force` 后缀的任务跳过确认
- 原始 ansible-playbook 命令仍然可用

### 配置同步安全机制

同步 `settings.json` 时会自动执行以下安全检查：

1. **差异检测**：使用 `jq -S` 对新旧配置进行排序格式化后比较
2. **差异预览**：显示统一 diff 格式的变更内容
3. **自动备份**：备份旧配置到 `tmps/backup/settings.json.YYYYMMDD_HHMMSS`
4. **二次确认**：人工确认后才执行更新（可通过 `confirm_settings_update` 禁用）

**配置项**：

```yaml
settings:
  confirm_settings_update: true # 设置为 false 可跳过确认（CI/CD 用）
```

**跳过确认的快捷命令**：

- `task sync-force` - 同步配置不询问
- `task deploy-force` - 完整部署不询问

### 修改配置流程

1. **修改变量**：编辑 `inventory/default/group_vars/all/settings.yml`
2. **测试渲染**：运行 `task check-sync`（或 `ansible-playbook playbooks/setup.yml --tags sync_config --check --diff`）
3. **执行同步**：运行 `task sync`（或 `ansible-playbook playbooks/setup.yml --tags sync_config`）
4. **验证结果**：运行 `task view-settings` 检查 `~/.claude/settings.json`

### 添加新的输出风格

1. 在 `claude-assets/output-styles/` 创建新的 `.md` 文件
2. 编写人格化指令（参考现有风格文件）
3. 运行同步命令：`ansible-playbook playbooks/setup.yml --tags sync_config`
4. 在 `settings.yml` 中修改 `outputStyle` 为新风格的文件名（不含 .md）

### 添加新的自定义命令

1. 在 `claude-assets/skills/<skill-name>/` 创建目录，添加 `SKILL.md` 文件
2. 按照 Claude 命令规范编写 front-matter 和指令内容
3. 运行同步命令：`ansible-playbook playbooks/setup.yml --tags sync_config`
4. 使用命令：`/<skill-name>`

**Front-matter 必需字段**：

```yaml
name: skill-name
description: 命令的简短描述（一句话）
allowed-tools: 允许使用的工具列表（如 Read(**), Bash(**)）
argument-hint: 参数提示（如 [--option] <required>）
```

**示例**：参考现有 skills `claude-assets/skills/ms-git-commit/SKILL.md` 等

### 常见问题排查

#### 1. Ansible 执行失败

**问题**：`ansible-playbook` 命令报错 "command not found"

**解决**：使用 `uv run` 前缀执行命令（项目使用 uv 管理依赖）

```bash
uv run ansible-playbook playbooks/setup.yml --tags sync_config
```

#### 2. Jinja2 渲染失败

**问题**：`settings.yml.j2` 渲染时报错 "undefined variable"

**排查步骤**：

1. 检查 `inventory/default/group_vars/all/settings.yml` 是否定义了相关变量
2. 检查 `secrets.yml` 是否包含敏感信息变量
3. 使用 `--check` 模式预览渲染结果：

```bash
uv run ansible-playbook playbooks/setup.yml --tags sync_config --check --diff
```

#### 3. rsync 同步失败

**问题**：文件未成功同步到 `~/.claude/` 目录

**排查步骤**：

1. 检查目标目录权限：`ls -la ~/.claude/`
2. 查看 Ansible 日志：`tail -f tmps/ansible.log`
3. 手动测试 rsync：`rsync -av claude-assets/skills/ ~/.claude/skills/`

#### 4. 插件安装失败

**问题**：插件安装步骤执行失败

**排查步骤**：

1. 检查 Claude CLI 是否已安装：`which claude`
2. 手动安装插件测试：`claude plugin install playwright@claude-plugins-official`
3. 查看已安装插件：`claude plugin list`

---

## 全局开发规范

### 代码风格

- **YAML 格式**：使用 2 空格缩进，键值对使用 `key: value` 格式
- **Jinja2 模板**：变量使用 `{{ variable }}` 格式，清晰注释变量来源
- **Markdown 文档**：遵循 CommonMark 规范，使用中文标点符号

### 文件命名约定

- **Playbook**：使用 `snake_case.yml` 命名（如 `sync_claude_config.yml`）
- **变量文件**：使用 `snake_case.yml` 命名（如 `settings.yml`）
- **自定义命令**：使用 `kebab-case.md` 命名（如 `git-commit.md`）
- **输出风格**：使用 `kebab-case.md` 命名（如 `nekomata-engineer.md`）

### 变量管理规范

- **公开配置**：放在 `settings.yml`（模型名称、API Base URL、输出风格等）
- **敏感信息**：放在 `secrets.yml`（API Key、密码等），并添加到 `.gitignore`
- **变量命名**：使用 `settings.` 前缀表示公开配置，`secrets.` 前缀表示敏感信息

### Markdown 文档规范

- `npm install -g markdownlint-cli` - 安装 Markdown 格式检查工具
- `markdownlint "**/*.md"` - 检查所有 Markdown 文件格式
- `.markdownlint.json` - 配置文件，已放宽 MD013 行长度至 120（中文文档友好）
- **Front-matter 陷阱**：YAML front-matter 中不要使用 `#` 注释，会导致解析失败
- **代码块规范**：所有代码块必须指定语言（如 ` ```text `、` ```bash `）
- **标题规范**：不要用加粗代替标题，使用 `###` 或引用块 `>`

### 调试技巧

- `cat -A <file>` - 查看文件的精确字符（包括全角/半角、隐藏字符）
- `od -c <file>` - 以八进制显示文件字节内容（用于调试编码问题）
- `sed -i 's/pattern/replacement/' <file>` - 批量替换文本（支持正则表达式）

### 文档维护规范

- **CLAUDE.md 定位**：开发指南（架构、规范、工具），而非项目历史或变更日志
- **禁止内容**：变更记录（用 git log）、大型 Mermaid 图（信息重复且占 token）、已废弃的命令/文件引用
- **定期验证**：运行 `grep -o 'playbooks/[a-z_-]*\.yml' CLAUDE.md | sort -u | xargs -I {} test -f {} || echo "Missing: {}"` 检查引用文件是否存在
- **精简优先级**：删除重复信息 > 删除过时内容 > 压缩冗余说明 > 链接到详细文档

### Ansible Playbook 组织模式

- 本项目使用**单一入口 playbook**（`setup.yml`）+ tags 控制执行阶段
- 不要创建独立的 `install_*.yml`，统一通过 `setup.yml --tags <stage>` 执行
- 可用 tags：`install_cli`、`install_plugins`、`sync_config`、`verify`
- `install_cli` tag 为兼容保留；当前仅检查 Claude CLI，并在需要时安装 ccline
- 查看所有 tags：`grep -E '^\s+tags:' playbooks/setup.yml | sort -u`

### Git 提交规范

遵循 Conventional Commits 规范：

- `feat(scope): 新增功能`
- `fix(scope): 修复缺陷`
- `docs(scope): 文档更新`
- `refactor(scope): 代码重构`
- `chore(scope): 杂务维护`

---

## 全局开发指导（补充）

### 实施前强制检查清单

**注意：在编写任何代码之前，必须完成此清单：**

#### 这是代码实施任务吗？

检查用户请求是否包含以下任一关键词：
- `实现`, `添加`, `写`, `创建`, `开发`, `修改`, `重构`
- `implement`, `add`, `write`, `create`, `develop`, `modify`, `refactor`

#### 如果是，必须按以下顺序执行：

1. **[ ] 先写设计文档**
   - 在 `docs/design/FEATURE_NAME.md` 中创建设计文档
   - **保持简洁**：目标 3-5 页，5 分钟内可读
   - **关注决策而非细节**：
     - 问题陈述和目标
     - 核心架构（带图表）
     - 关键设计决策
     - 需要用户输入的开放问题
   - **不要过度记录**：
     - 无详细代码示例（留给实施阶段）
     - 无明显的实施细节
     - 无详尽的 API 文档（留给代码注释）

2. **[ ] 获取用户批准**
   - 使用 AskUserQuestion 工具展示设计
   - 询问："我已完成设计文档。是否继续实施？"
   - 等待明确批准

3. **[ ] 编写测试**
   - 在 `tests/unit/`、`tests/integration/` 或适当的测试目录中创建测试文件
   - 遵循红-绿-重构循环

4. **[ ] 实施代码**
   - **只有现在**才能编写实施代码
   - 编写最少代码来通过测试

5. **[ ] 验证**
   - 运行测试
   - 验证实施与设计匹配
   - 如需要更新文档

#### 如果你跳过上述任何步骤，就是违反项目规则。

**例外**：简单任务如修复拼写错误、读取文件或回答问题不需要此清单。

---

### 开发工作流：文档优先 + TDD

此工作流遵循**严格的文档优先 TDD**：

1. **文档优先** - 在编写代码前更新架构文档
2. **编写测试** - 遵循红-绿-重构循环
3. **实施** - 编写最少代码来通过测试
4. **验证** - 确保实施与记录的设计匹配

**永远不要跳过文档。永远不要跳过测试。**

---

### 文档标准

#### 推荐目录结构

```
docs/
├── architecture/      # 系统级架构文档 (*_arch.md)
├── design/           # 功能设计文档 (*_design.md)
├── adr/              # 架构决策记录 (###_topic.md)
├── guides/           # 使用指南 (*_guide.md)
├── integration/      # 集成文档 (*_integration.md)
├── api/              # API 文档 (*_api.md)
└── references/       # 参考材料
```

#### 命名规范

**核心原则**：关键词优先 + 后缀标识符（便于 AI 搜索）

| 文档类型 | 格式 | 示例 |
|----------|--------|---------|
| 架构 | `{topic}_arch.md` | `orchestrator_arch.md` |
| 设计 | `{feature}_design.md` | `streaming_thought_design.md` |
| ADR | `{number}_{topic}.md` | `001_single_vs_multi_agent.md` |
| 指南 | `{topic}_guide.md` | `configuration_guide.md` |
| 集成 | `{system}_integration.md` | `gateway_integration.md` |

#### 文档搜索策略

**按主题搜索（推荐）**：
```python
# 查找与主题相关的所有文档
Glob("docs/**/*orchestrator*")
Glob("docs/**/*streaming*")
```

**按类型搜索**：
```python
# 查找所有架构文档
Glob("docs/**/*_arch.md")

# 查找所有设计文档
Glob("docs/**/*_design.md")

# 查找所有指南
Glob("docs/**/*_guide.md")
```

**按目录搜索**：
```python
Glob("docs/architecture/*.md")
Glob("docs/adr/*.md")
```

#### 搜索最佳实践

1. **首先确定文档类型**：
   - 组件架构 → `docs/architecture/`
   - 功能设计 → `docs/design/`
   - 决策背景 → `docs/adr/`
   - 操作指南 → `docs/guides/`
   - 外部集成 → `docs/integration/`

2. **使用精确的 Glob 模式**：
   - ✅ `Glob("docs/architecture/*orchestrator*")` - 精确
   - ⚠️ `Glob("docs/**/*orchestrator*")` - 全局搜索（后备方案）
   - ❌ `Glob("docs/**/*.md")` - 太宽泛

3. **先读取索引文件**：
   - 每个目录应该有一个 `README.md` 作为索引
   - 索引提供快速导航和概览

4. **当文档不存在时**：
   - 检查 `docs/README.md` 确认是否应该存在
   - 遵循命名规范创建新文档

5. **避免冗余搜索**：
   - 不要同时使用 Glob 和 Grep 搜索相同内容
   - 使用 Glob 定位文件，然后用 Read 查看内容

---

### 设计文档原则

编写设计文档时：

1. **保持简洁**：3-5 页，5 分钟内可读
2. **关注决策**：做什么和为什么，而非每个细节
3. **包含**：
   - 问题陈述和目标
   - 核心架构（如有帮助带图表）
   - 关键设计决策和权衡
   - 需要用户输入的开放问题
4. **排除**：
   - 详细代码示例（留给实施阶段）
   - 明显的实施细节
   - 详尽的 API 文档（属于代码注释）

---

### 代码质量原则

- **避免过度工程**：只进行直接请求或明确必要的更改
- **保持方案简单**：不要添加超出要求的功能、重构或"改进"
- **最小化代码**：编写当前任务所需的最少代码
- **删除未使用的代码**：不要留下向后兼容的黑客或未使用的变量
- **安全第一**：避免 OWASP 前 10 大漏洞（XSS、SQL 注入等）
- **持续加固**：安全不是一次性任务；每次更改都应用增量加固

---

### Git 提交标准：原子提交 + Conventional Commits

#### 原子提交

每个提交应该**只做一件事**。好处：
- `git bisect` 可以快速定位问题
- 回滚安全且不影响无关功能
- 代码审查清晰且专注

**不好的示例**：
```
feat: add user auth, fix navbar, update docs
```

**好的示例**：
```
feat: add JWT token validation middleware
fix: correct navbar alignment on mobile
docs: update auth setup guide
```

#### Conventional Commits

所有提交必须使用类型前缀：

| 类型 | 用途 | 示例 |
|------|---------|---------|
| `feat:` | 新功能 | `feat: add dark mode toggle` |
| `fix:` | 错误修复 | `fix: resolve memory leak in WebSocket handler` |
| `docs:` | 仅文档 | `docs: update API endpoint reference` |
| `chore:` | 构建、依赖、配置 | `chore: upgrade vite to v6` |
| `test:` | 添加或更新测试 | `test: add unit tests for auth middleware` |
| `refactor:` | 代码重构，无行为变更 | `refactor: extract validation logic to shared util` |
| `style:` | 格式，无逻辑变更 | `style: fix indentation in config file` |
| `perf:` | 性能改进 | `perf: cache database query results` |

#### 提交纪律

- `fix` 提交不得更改 API 契约
- `feat` 提交必须有明确的范围边界
- `refactor` 提交不得更改可观察行为
- `docs` 提交必须独立于代码更改
- 每个功能更改应伴随相关的 `docs` 和 `test` 提交

---

### CLI 优先工具原则

选择工具和服务时，优先考虑具有 CLI 接口的工具：
- CLI 工具是确定的、可测试的且文档完善的
- AI 代理可以直接调用它们，无需额外的抽象层
- 在项目 CLAUDE.md 中列出可用的 CLI 工具（例如 `logs: axiom or vercel cli`）
- 避免在 CLI 工具上进行不必要的包装或抽象层

CLI 友好工具示例：`gh`、`psql`、`vercel`、`docker`、`kubectl`、`aws`、`gcloud`

---

### CLAUDE.md 编写原则

此文件本身应遵循以下规则：

1. **保持简洁**：前沿模型可以可靠地遵循约 150-200 条指令。每一行都应该证明自己的价值——问"删除这会导致错误吗？"如果不会，就删掉它。
2. **渐进式披露**：不要倾倒所有可能的信息。告诉 Claude *如何找到*信息，而不是预先列出所有内容。使用引用如"详见 `docs/architecture/`"。
3. **可操作胜于信息性**：优先考虑"使用 ES 模块，而非 CommonJS"而非"项目使用 ES 模块"。
4. **定期更新**：像对待代码一样对待 CLAUDE.md——当出现问题时审查它，定期精简它。
5. **适当分层**：
   - `~/.claude/CLAUDE.md` — 全局偏好（本文件）
   - `PROJECT_ROOT/CLAUDE.md` — 共享团队上下文
   - `subdirectory/CLAUDE.md` — 模块特定指导
   - `CLAUDE.local.md` — 个人覆盖（git 忽略）

---

## AI 使用指引

### 自定义 Skills

本项目提供以下自定义命令：

- `/ms-git-commit [--emoji] [--no-verify]`：智能分析 Git 改动并生成 Conventional Commits 风格的提交信息
- `/init-project <项目摘要>`：初始化项目 AI 上下文，生成根级与模块级 CLAUDE.md 索引
- `/my-claude-sync-upstream`：专门用于 my-claude 项目从上游仓库一键同步更新，自动检测并添加 upstream remote，处理配置迁移

### 自定义智能体

- **init-architect**：自适应初始化项目架构文档，支持分阶段扫描与增量更新
- **get-current-datetime**：获取当前时间戳（用于文档生成）

### 输出风格

可在 `settings.yml` 中切换以下输出风格：

- `engineer-professional`：专业工程师风格（标准、简洁）
- `nekomata-engineer`：猫娘工程师风格（可爱、活泼）
- `ojousama-engineer`：大小姐工程师风格（优雅、礼貌）
- `laowang-engineer`：老王工程师风格（幽默、接地气）

修改后运行 `ansible-playbook playbooks/setup.yml --tags sync_config` 生效。

### 模型配置

在 `settings.yml` 中配置多层级模型：

```yaml
settings:
  env:
    ANTHROPIC_DEFAULT_OPUS_MODEL: "claude-4.6-opus" # 用于 Opus 或计划模式
    ANTHROPIC_DEFAULT_SONNET_MODEL: "claude-4.5-sonnet" # 用于 Sonnet（默认）
    ANTHROPIC_DEFAULT_HAIKU_MODEL: "claude-4.6-haiku" # 用于后台功能
    CLAUDE_CODE_SUBAGENT_MODEL: "claude-4.5-sonnet" # 用于子代理
```

### Claude 插件管理

#### 查看可用插件

官方市场（`claude-plugins-official`）提供多个插件，包括：

- `playwright`（浏览器自动化）、`serena`（语义代码分析）、`context7`（文档查询）
- `github`/`gitlab`（代码托管平台）、`slack`/`linear`/`asana`（协作工具）
- `firebase`/`supabase`（后端服务）、`stripe`（支付集成）

**完整列表**：运行 `claude plugin list --available` 或查看 `inventory/default/group_vars/all/settings.yml` 中的 `enabled_plugins`

#### 启用插件

##### 方法 1：仅更新配置（推荐，适合已安装插件）

1. 编辑 `inventory/default/group_vars/all/settings.yml`
2. 在 `enabled_plugins` 列表中添加插件名称：

   ```yaml
   settings:
     enabled_plugins:
       - "playwright@claude-plugins-official"
       - "context7@claude-plugins-official"
       - "serena@claude-plugins-official"
   ```

3. 运行 `uv run ansible-playbook playbooks/setup.yml --tags sync_config`

##### 方法 2：自动安装并启用插件（推荐，适合新插件）

1. 编辑 `inventory/default/group_vars/all/settings.yml`
2. 在 `enabled_plugins` 列表中添加插件名称
3. 运行 `uv run ansible-playbook playbooks/setup.yml`（自动安装并同步）

**特性说明**：

- `setup.yml` 支持幂等性：已安装的插件不会重复安装
- 自动检测缺失插件，仅安装未安装的插件
- 安装完成后自动验证插件可用性

#### 禁用插件

1. 编辑 `settings.yml`，从 `enabled_plugins` 列表中移除插件名称
2. 运行 `uv run ansible-playbook playbooks/setup.yml --tags sync_config`

#### 卸载插件

```bash
claude plugin uninstall <插件名>@claude-plugins-official
```

#### 查看已安装插件

```bash
claude plugin list
```

---

### 自定义 MCP 服务器管理

#### 什么是自定义 MCP 服务器？

- **Claude 插件**：通过 `claude plugin install` 安装，由官方市场提供，插件内部包含 `.mcp.json` 定义
- **自定义 MCP 服务器**：不在官方市场，使用 `claude mcp add-json` 命令管理

#### ⚠️ 重要注意事项

- **不要**在 `settings.json` 中配置 `mcpServers`（该配置项不在官方 schema 中）
- MCP 配置使用 `claude mcp` 命令管理，或通过项目根目录 `.mcp.json`（项目级别）或 `~/.claude.json`（用户级别）

#### 添加自定义 MCP 服务器

1. 编辑 `inventory/default/group_vars/all/settings.yml`
2. 在 `custom_mcp_servers` 列表中添加服务器定义（与 `settings` 平级）：

   ```yaml
   claude_json:
     # 同步控制（可选，默认 true）
     confirm_claude_json_update: true
     mcpServers:
       my-custom-mcp:
         type: "stdio" # 可选，默认 stdio
         command: "python"
         args: ["/path/to/server.py"]
         env:
           API_KEY: "your-api-key"
   ```

3. 运行 `task sync-claude-json` 或 `uv run ansible-playbook playbooks/setup.yml --tags sync_claude_json`

#### 配置说明

**通用参数**：

- `name`：MCP 服务器名称（唯一标识）
- `type`：服务器类型（`stdio`/`http`/`sse`，默认 `stdio`）

**stdio 类型参数**：

- `command`：启动命令（如 python、node、npx、uvx）
- `args`：命令参数（数组格式）
- `env`：环境变量（可选，支持从 secrets.yml 引用）

**http/sse 类型参数**：

- `url`：服务器 URL
- `headers`：HTTP 请求头（用于认证）

#### 示例配置

**Python MCP 服务器（stdio）**：

```yaml
custom_mcp_servers:
  - name: "python-mcp"
    type: "stdio"
    command: "python"
    args: ["/home/user/mcp-server/server.py"]
```

**HTTP MCP 服务器**：

```yaml
custom_mcp_servers:
  - name: "remote-api"
    type: "http"
    url: "https://mcp.example.com/api"
    headers:
      Authorization: "Bearer {{ secrets.remote_mcp_api_key }}"
```

**通过 npx 启动**：

```yaml
custom_mcp_servers:
  - name: "npx-mcp"
    command: "npx"
    args: ["-y", "@company/mcp-server"]
```

**引用敏感信息**：

```yaml
# settings.yml
custom_mcp_servers:
  - name: "secure-mcp"
    command: "python"
    args: ["/path/to/server.py"]
    env:
      API_KEY: "{{ secrets.custom_mcp_api_key }}"

# secrets.yml
secrets:
  custom_mcp_api_key: "your-secret-key-here"
```

#### Claude Code MCP 常用命令

```bash
# 列出已配置的 MCP 服务器
claude mcp list

# 使用 JSON 添加 MCP 服务器
claude mcp add-json --scope user my-server '{"type":"stdio","command":"npx","args":["server"]}'

# 移除 MCP 服务器
claude mcp remove my-server
```

---

## CI/CD 集成

### GitHub Actions 自动化部署

本项目提供 GitHub Actions workflow，可在全新机器上自动化测试部署流程。

#### CI/CD 快速开始

直接推送即可触发自动部署测试（无需配置）：

```bash
git push origin master  # 推送到 master/main/develop 自动触发
```

**可选配置**：在仓库 Settings → Secrets 中添加 `ANTHROPIC_API_KEY` 测试真实 API

**详细文档**：[.github/workflows/README.md](.github/workflows/README.md)

---

## 许可证

本项目采用 [MIT License](LICENSE) 开源
