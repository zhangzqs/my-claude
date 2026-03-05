---
name: my-claude-sync-upstream
description: 专门用于 my-claude 项目从上游仓库一键同步更新。自动检测并添加 upstream remote，从 upstream/master 更新到本地 master，合并到当前分支，检查配置变更并提示调整 inventory 配置，最后引导提交和推送。当用户在 my-claude 项目中说"从上游更新"、"同步上游"、"拉取上游代码"等时使用此 skill。
allowed-tools: Bash(git, cat, ls, diff, uv), Read(**), Edit(**)
---

# my-claude 项目上游同步 Skill

专门用于 my-claude 项目从上游仓库一键同步更新的完整流程。

---

## 功能流程

执行以下完整步骤：

### 1. 检查环境

- 确认当前在 git 仓库中
- 确认这是 my-claude 项目（检查是否有 ansible.cfg、inventory/、claude-assets/ 等目录）

### 2. 检查并添加 upstream remote

- 运行 `git remote -v` 查看现有 remotes
- 如果没有 `upstream`，询问用户上游仓库地址，然后执行：
  ```bash
  git remote add upstream <upstream-url>
  ```
- 默认建议上游为 `git@github.com:zhangzqs/my-claude.git` 或 `https://github.com/zhangzqs/my-claude.git`

### 2.5 检测是否就在上游仓库本身

- 如果检测到 `origin` 指向 `zhangzqs/my-claude`，且没有配置 `upstream` remote，或者用户就在上游仓库中开发，则：
  - 终止流程，告知用户："检测到当前仓库本身就是上游仓库（origin 指向 zhangzqs/my-claude），无需从上游同步更新。这个 skill 是给从上游 fork 出来的用户使用的。"
  - 不执行任何 git 操作

### 3. 保存当前分支

- 记录当前工作分支名称

### 4. 更新本地 master

```bash
git checkout master
git fetch upstream
git merge upstream/master
```

### 5. 合并到当前分支

```bash
git checkout <current-branch>
git merge master
```

### 6. 处理冲突

如果发生冲突：

- 列出冲突文件
- 对于 ansible.cfg，保留用户的 inventory 配置（指向用户个人 inventory 目录）
- 对于其他文件，询问用户如何处理，或提供差异预览
- 解决后执行 `git add` 和 `git merge --continue`

### 7. 检查与迁移配置变更

#### 7.1 对比配置

- 读取上游的 `inventory/default/group_vars/all/settings.yml`
- 读取用户的 `inventory/<username>/group_vars/all/settings.yml`
- 对比两者的结构差异

#### 7.2 自动迁移配置（处理破坏性变更）

**新增配置项**：

- 上游有但用户配置没有的，自动添加到用户配置中，使用上游的默认值
- 向用户报告："新增配置项 X，已使用默认值 Y 添加到你的配置"

**删除配置项**：

- 用户配置有但上游已删除的，保留在用户配置中但标注为 deprecated
- 向用户报告："配置项 X 在上游已移除，你的配置中仍保留，是否删除？"

**结构重构（破坏性变更）**：

- 如果上游配置结构发生重大变化（如变量从 `settings.xxx` 移动到 `settings.yyy.xxx`）：
  - 尝试自动迁移：读取用户旧配置的值，写入新结构中
  - 保留旧配置作为注释或备份
  - 向用户报告迁移详情
  - 如果无法自动迁移（结构完全不兼容），终止流程并询问用户

**值更新**：

- 上游配置项的值发生变化（如默认模型、插件列表等）：
  - 询问用户："配置项 X 的值从 Y 变为 Z，是否更新？（Y=保留你的值，Z=使用上游新值）"
  - 默认保留用户的值

#### 7.3 其他配置文件

同时检查并迁移：

- `inventory/default/group_vars/all/models.yml` → `inventory/<username>/group_vars/all/models.yml`
- `inventory/default/group_vars/all/mcp_servers.yml` → `inventory/<username>/group_vars/all/mcp_servers.yml`
- `inventory/default/group_vars/all/secrets.yml.example` → 提示用户检查是否有新的敏感配置需要添加

### 8. 收尾与指引

- 显示更新摘要：从上游拉取了多少 commit，是否有冲突
- 提示用户：
  - 检查配置是否正常
  - 运行 `task check` 预览变更
  - 提交和推送

---

## 执行风格

- 清晰地向用户报告每一步在做什么
- 遇到需要用户决策的地方，明确列出选项
- 对于 my-claude 项目的特定文件（ansible.cfg、inventory 配置）要有特殊处理逻辑
