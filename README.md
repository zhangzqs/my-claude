# my-claude-ansible

<div align="center">

## Claude Code + Codex CLI 配置管理仓库

基于 Ansible 的声明式配置管理,实现 Claude Code 和 Codex CLI 个性化配置的自动化部署与同步

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Ansible](https://img.shields.io/badge/Ansible-2.9%2B-blue)](https://www.ansible.com/)
[![Python](https://img.shields.io/badge/Python-3.8%2B-green)](https://www.python.org/)

</div>

---

## 特性

- **声明式配置管理**：使用 YAML 声明期望状态,由 Ansible 自动执行
- **多工具统一管理**：同时管理 Claude Code (`~/.claude/`) 和 Codex CLI (`~/.codex/`) 的配置
- **个性化输出风格**：支持多种人格化输出风格(猫娘工程师、大小姐工程师等)
- **自定义命令扩展**：通过 Markdown 定义 Claude 自定义命令(如 `/git-commit`、`/init-project`)
- **智能体编排**：支持子 Agent 定义,实现复杂任务分解与自动化
- **插件管理**：一键安装与配置 Claude 官方插件(Playwright、Serena、Context7 等)
- **安全配置**：敏感信息与公开配置分离,支持 Ansible Vault 加密
- **CI/CD 集成**：提供 GitHub Actions 自动化测试流程

---

## 快速开始

### 前置条件

- Claude CLI（需用户预先安装，例如执行：`curl -fsSL https://claude.ai/install.sh | bash`）
- Python 3.8+
- Ansible 2.9+(推荐通过 uv 管理)
- Node.js 18+(用于安装 ccline)

### 一键部署

```bash
# 1. 克隆仓库
git clone https://github.com/yourusername/my-claude-ansible.git
cd my-claude-ansible

# 2. 安装 Claude CLI（需用户自行完成）
curl -fsSL https://claude.ai/install.sh | bash

# 3. 配置敏感信息
cp inventory/default/group_vars/all/secrets.yml.example \
   inventory/default/group_vars/all/secrets.yml
# 编辑 secrets.yml,填入你的 ANTHROPIC_API_KEY

# 4. 自定义配置(可选)
vim inventory/default/group_vars/all/settings.yml

# 5. 一键部署
uv run ansible-playbook playbooks/setup.yml
```

部署前请确保 `claude` 命令已经可用；playbook 只负责检查，不会自动安装 Claude CLI。

**部署完成后**:

- Claude 配置已同步到 `~/.claude/settings.json`
- 自定义命令已安装到 `~/.claude/skills/`
- 输出风格已同步到 `~/.claude/output-styles/`
- Codex CLI 配置已同步到 `~/.codex/config.toml`
- Codex 指令文件已同步到 `~/.codex/AGENTS.md`

---

## 多人使用流程

如果你是从上游仓库 Fork 出来的用户，可以按照以下流程配置和使用。

### 初始化流程（以下游用户 alice 为例）

```bash
# 1. Fork 上游仓库（GitHub 操作）
# 访问 https://github.com/zhangzqs/my-claude，点击右上角的 Fork 按钮

# 2. Clone 你的 fork 到本地
git clone git@github.com:alice/my-claude.git
cd my-claude

# 3. 添加上游仓库（用于后续同步更新）
git remote add upstream git@github.com:zhangzqs/my-claude.git

# 4. 创建你的个人分支
git checkout -b alice

# 5. 创建你的 inventory 目录（复制 default 作为模板）
cp -r inventory/default inventory/alice

# 6. 修改 ansible.cfg 指向你的 inventory
# 编辑第 28 行，将 inventory 路径改为：
# inventory = $PWD/inventory/alice/inventory.yml

# 7. 配置你的个人设置
# 编辑 inventory/alice/group_vars/all/settings.yml
# 编辑 inventory/alice/group_vars/all/secrets.yml（复制 secrets.yml.example）

# 8. 提交你的个人配置
git add ansible.cfg inventory/alice/
git commit -m "chore: 添加 alice 个人配置"
git push -u origin alice
```

### 从上游更新流程

```bash
# 1. 切换到 master 分支
git checkout master

# 2. 从上游仓库拉取最新代码
git fetch upstream
git merge upstream/master

# 3. 切换回你的个人分支
git checkout alice

# 4. 将 master 分支合并到你的分支
git merge master

# 5. 如果 ansible.cfg 有冲突，解决冲突（保留你的 inventory 路径配置）

# 6. 检查配置变更，根据需要调整 inventory/alice/ 下的配置文件

# 7. 提交并推送更新
git add .
git commit -m "chore: 同步上游更新"
git push
```

#### 使用项目级 skill 一键更新（推荐）

在 my-claude 项目目录下，直接使用项目内置 skill：

```bash
/my-claude-sync-upstream
```

这个命令会自动完成上述所有步骤，包括配置迁移。

---

## 文档

- **[完整文档](CLAUDE.md)**：项目架构、开发指南、API 参考
- **[快速开始](CLAUDE.md#快速开始)**：一键部署流程
- **[自定义命令](CLAUDE.md#自定义命令)**：使用 `/git-commit`、`/init-project` 等命令
- **[插件管理](CLAUDE.md#claude-插件管理)**：安装与配置 Claude 插件
- **[CI/CD 集成](CLAUDE.md#cicd-集成)**：GitHub Actions 自动化部署
- **[故障排查](CLAUDE.md#常见问题排查)**：常见问题解决方案

---

## 输出风格

支持以下人格化输出风格(在 `settings.yml` 中配置 `outputStyle`):

| 风格                    | 特点                 | 适用场景           |
| ----------------------- | -------------------- | ------------------ |
| `engineer-professional` | 专业、简洁、标准     | 正式项目、技术文档 |
| `nekomata-engineer`     | 可爱、活泼、猫娘     | 轻松氛围、个人项目 |
| `ojousama-engineer`     | 优雅、礼貌、大小姐   | 注重礼节、高雅场景 |
| `laowang-engineer`      | 幽默、接地气、老王   | 团队协作、日常开发 |
| `marionette-engineer`   | 冷峻、理性、知识至上 | 严谨研究、架构设计 |

---

## 自定义命令

### `/git-commit` - 智能提交命令

自动分析 Git 改动并生成 Conventional Commits 风格的提交信息:

```bash
/git-commit                           # 分析当前改动
/git-commit --all                     # 暂存所有改动并提交
/git-commit --emoji                   # 使用 Emoji 提交信息
/git-commit --type feat --scope ui    # 指定类型和作用域
```

### `/init-project` - 项目初始化命令

自动生成根级与模块级 CLAUDE.md 索引:

```bash
/init-project my-project 项目描述
```

---

## 插件管理

### 支持的官方插件

| 插件       | 功能描述                       |
| ---------- | ------------------------------ |
| playwright | 浏览器自动化与端到端测试       |
| serena     | 语义代码分析与重构建议         |
| context7   | 最新文档查询(从源仓库拉取)     |
| github     | GitHub 仓库管理(Issues、PR 等) |
| gitlab     | GitLab DevOps 平台集成         |
| slack      | Slack 工作区集成               |
| linear     | Linear Issue 跟踪集成          |

### 安装插件

编辑 `inventory/default/group_vars/all/settings.yml`:

```yaml
settings:
  enabled_plugins:
    - "playwright@claude-plugins-official"
    - "context7@claude-plugins-official"
    - "serena@claude-plugins-official"
```

运行部署命令:

```bash
uv run ansible-playbook playbooks/setup.yml --tags install_plugins  # 安装插件
uv run ansible-playbook playbooks/setup.yml --tags sync_config  # 同步配置
```

---

## Codex CLI 配置

本项目同时管理 OpenAI Codex CLI 的配置,与 Claude Code 并行部署、互不干扰,共享 API keys。

### 配置文件对照

| Codex 概念 | 对应 Claude 概念 | 部署位置 |
|-----------|-----------------|---------|
| `config.toml` | `settings.json` | `~/.codex/config.toml` |
| `AGENTS.md` | `CLAUDE.md` | `~/.codex/AGENTS.md` |

### 配置变量

编辑 `inventory/<config>/group_vars/all/codex_config.yml`:

```yaml
codex_config:
  model_provider: "ppio"
  model: "pa/gpt-5.4"
  model_reasoning_effort: "xhigh"
  sandbox_mode: "read-only"

codex_model_providers:
  ppio:
    base_url: "https://api.ppio.com/openai/v1"
    wire_api: "responses"
    api_key: "{{ secrets.api_keys.jdo_key }}"
```

### 常用命令

```bash
task sync-codex         # 同步 Codex 配置
task check-codex        # 预览 Codex 配置变更
task view-codex-config  # 查看当前 Codex 配置
```

---

## 开发指南

### 添加自定义命令

1. 在 `claude-assets/skills/mc/` 创建 `my-command.md`
2. 编写 front-matter 和指令内容:

```markdown
---
description: 我的自定义命令
allowed-tools: Read(**), Write(**)
---

# My Command

命令说明...
```

1. 同步配置:`uv run ansible-playbook playbooks/setup.yml --tags sync_config`

### 添加输出风格

1. 在 `claude-assets/output-styles/` 创建 `my-style.md`
2. 编写人格化指令
3. 在 `settings.yml` 中设置 `outputStyle: "my-style"`
4. 同步配置

---

## 贡献指南

欢迎提交 Issue 和 Pull Request!

**贡献方式**:

1. Fork 本仓库
2. 创建特性分支:`git checkout -b feature/my-feature`
3. 提交改动:`git commit -m "feat: 新增功能"`(遵循 Conventional Commits)
4. 推送分支:`git push origin feature/my-feature`
5. 提交 Pull Request

**代码规范**:

- 遵循 Conventional Commits 提交信息规范
- YAML 文件使用 2 空格缩进
- 新增功能需更新文档

---

## 许可证

本项目采用 [MIT License](LICENSE) 开源。

---

## 致谢

- [Anthropic Claude](https://www.anthropic.com/) - 强大的 AI 助手
- [OpenAI Codex](https://openai.com/) - 智能编程助手
- [Ansible](https://www.ansible.com/) - 优雅的自动化工具
- 所有贡献者

---

## 联系方式

- **Issues**:[GitHub Issues](https://github.com/yourusername/my-claude-ansible/issues)
- **Discussions**:[GitHub Discussions](https://github.com/yourusername/my-claude-ansible/discussions)

---

<div align="center">

**如果这个项目对你有帮助,请给个 Star!**

</div>
