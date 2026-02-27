# my-claude-ansible

> **Claude Code 配置管理仓库** - 基于 Ansible 的声明式配置管理，实现 Claude 个性化配置的自动化部署与同步

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
│   ├── commands/        # 自定义命令定义
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
4. 使用 rsync 同步 `commands`、`output-styles`、`CLAUDE.md` 等资源文件到 `~/.claude/`

---

## 模块索引

| 模块路径                       | 职责                                                      | 关键文件                                                 |
| ------------------------------ | --------------------------------------------------------- | -------------------------------------------------------- |
| `claude-assets/`               | Claude 配置资源仓库，包含模板、命令、智能体、输出风格定义 | `settings.yml.j2`, `CLAUDE.md`                           |
| `claude-assets/commands/`      | 自定义命令定义（Markdown 格式）                           | `git-commit.md`, `git-sync-branch.md`, `init-project.md` |
| `claude-assets/agents/`        | 自定义智能体定义（子 Agent）                              | `init-architect.md`, `get-current-datetime.md`           |
| `claude-assets/output-styles/` | 个性化输出风格定义（人格化）                              | `nekomata-engineer.md`, `laowang-engineer.md` 等         |
| `inventory/`                   | Ansible 清单与变量管理                                    | `inventory.yml`, `settings.yml`, `secrets.yml`           |
| `playbooks/`                   | Ansible Playbook 剧本                                     | `setup.yml`（通过 tags 控制执行阶段）                    |

---

## 快速开始

### 前置条件

- Python 3.8+
- Ansible 2.9+（推荐通过 uv 管理）
- Node.js 18+（用于安装 Claude CLI）

**命令执行说明**：

- 本项目使用 uv 管理 Python 依赖，所有 `ansible-playbook` 命令需添加 `uv run` 前缀
- 如果你已全局安装 Ansible，可直接使用 `ansible-playbook` 命令（不加前缀）

### 一键部署（推荐）

```bash
# 1. 克隆仓库
git clone <仓库地址> my-claude
cd my-claude

# 2. 配置变量
# 编辑 inventory/default/group_vars/all/settings.yml（公开配置）
vim inventory/default/group_vars/all/settings.yml
# 编辑 inventory/default/group_vars/all/secrets.yml（敏感信息）
vim inventory/default/group_vars/all/secrets.yml

# 3. 一键部署（安装插件 + 同步配置）
uv run ansible-playbook playbooks/setup.yml
```

**`setup.yml` 会自动执行**：

1. 检查 Claude CLI 是否已安装（未安装会提示）
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
ls -la ~/.claude/commands/

# 检查输出风格是否同步
ls -la ~/.claude/output-styles/
```

---

## 运行与开发

### 主要命令

```bash
# 一键部署（推荐）
uv run ansible-playbook playbooks/setup.yml

# 同步配置到 ~/.claude（常用）
uv run ansible-playbook playbooks/setup.yml --tags sync_config

# 仅安装插件
uv run ansible-playbook playbooks/setup.yml --tags install_plugins

# 仅检查 Claude CLI（不安装）
uv run ansible-playbook playbooks/setup.yml --tags install_cli --check

# 查看配置变量（不执行）
uv run ansible-playbook playbooks/setup.yml --tags sync_config --check --diff

# 验证 Ansible 配置
ansible-config dump --only-changed

# 查看 Ansible 日志
tail -f tmps/ansible.log
```

### 修改配置流程

1. **修改变量**：编辑 `inventory/default/group_vars/all/settings.yml`
2. **测试渲染**：运行 `ansible-playbook playbooks/setup.yml --tags sync_config --check --diff`
3. **执行同步**：运行 `ansible-playbook playbooks/setup.yml --tags sync_config`
4. **验证结果**：检查 `~/.claude/settings.json` 和相关文件

### 添加新的输出风格

1. 在 `claude-assets/output-styles/` 创建新的 `.md` 文件
2. 编写人格化指令（参考现有风格文件）
3. 运行同步命令：`ansible-playbook playbooks/setup.yml --tags sync_config`
4. 在 `settings.yml` 中修改 `outputStyle` 为新风格的文件名（不含 .md）

### 添加新的自定义命令

1. 在 `claude-assets/commands/mc/` 创建新的 `.md` 文件
2. 按照 Claude 命令规范编写 front-matter 和指令内容
3. 运行同步命令：`ansible-playbook playbooks/setup.yml --tags sync_config`
4. 使用命令：`/your-command-name`

**Front-matter 必需字段**：

```yaml
description: 命令的简短描述（一句话）
allowed-tools: 允许使用的工具列表（如 Read(**), Bash(**)）
argument-hint: 参数提示（如 [--option] <required>）
```

**示例**：参考现有命令 `claude-assets/commands/mc/git-commit.md` 或 `git-sync-branch.md`

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
3. 手动测试 rsync：`rsync -av claude-assets/commands/ ~/.claude/commands/`

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
- 查看所有 tags：`grep -E '^\s+tags:' playbooks/setup.yml | sort -u`

### Git 提交规范

遵循 Conventional Commits 规范：

- `feat(scope): 新增功能`
- `fix(scope): 修复缺陷`
- `docs(scope): 文档更新`
- `refactor(scope): 代码重构`
- `chore(scope): 杂务维护`

---

## AI 使用指引

### 自定义命令

本项目提供以下自定义命令：

- `/git-commit [--emoji] [--no-verify]`：智能分析 Git 改动并生成 Conventional Commits 风格的提交信息
- `/init-project <项目摘要>`：初始化项目 AI 上下文，生成根级与模块级 CLAUDE.md 索引

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
- **自定义 MCP 服务器**：不在官方市场，需要手动在 `settings.json` 中配置启动命令和参数

#### 添加自定义 MCP 服务器

1. 编辑 `inventory/default/group_vars/all/settings.yml`
2. 在 `custom_mcp_servers` 列表中添加服务器定义：

   ```yaml
   settings:
     custom_mcp_servers:
       - name: "my-custom-mcp"
         command: "python"
         args:
           - "/path/to/server.py"
         env:
           API_KEY: "your-api-key"
   ```

3. 运行 `ansible-playbook playbooks/setup.yml --tags sync_config`

#### 配置说明

- `name`：MCP 服务器名称（在 Claude 中显示）
- `command`：启动命令（如 python、node、npx、uvx）
- `args`：命令参数（数组格式）
- `env`：环境变量（可选，支持从 secrets.yml 引用敏感信息）

#### 示例配置

**Python MCP 服务器**：

```yaml
custom_mcp_servers:
  - name: "python-mcp"
    command: "python"
    args: ["/home/user/mcp-server/server.py"]
```

**Node.js MCP 服务器**：

```yaml
custom_mcp_servers:
  - name: "node-mcp"
    command: "node"
    args: ["/home/user/mcp-server/index.js"]
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
