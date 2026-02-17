# inventory 模块

[根目录](../CLAUDE.md) > **inventory**

---

> **Ansible 清单与变量管理** - 管理目标主机清单、环境变量和配置参数

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

`inventory/` 目录负责 Ansible 的清单与变量管理，提供：

1. **主机清单管理**：定义 Ansible 操作的目标主机（本项目使用 `localhost`）
2. **变量分层管理**：通过 `group_vars` 和 `host_vars` 实现变量的层级化组织
3. **环境隔离**：支持多环境配置（`default`、`dev`、`prod` 等）
4. **敏感信息分离**：将公开配置（`settings.yml`）与敏感信息（`secrets.yml`）分离存储

**核心理念**：

- **声明式配置**：所有配置变量以 YAML 声明，由 Ansible 自动应用
- **版本控制友好**：敏感信息独立存储，可选择性提交到版本控制
- **环境一致性**：确保开发、测试、生产环境的配置一致

---

## 目录结构

```text
inventory/
└── default/                       # 默认环境
    ├── inventory.yml              # 主机清单
    └── group_vars/                # 组变量目录
        └── all/                   # 应用于所有主机的变量
            ├── settings.yml       # 公开配置变量
            └── secrets.yml        # 敏感配置变量（API Key 等）
```

**目录说明**：

- `default/`：默认环境配置（可扩展为 `dev/`、`prod/` 等多环境）
- `group_vars/all/`：应用于所有主机的变量（本项目只有 localhost）
- `settings.yml`：公开配置，可提交到 Git
- `secrets.yml`：敏感配置，应添加到 `.gitignore`

---

## 关键文件说明

### 1. `inventory.yml`

**用途**：定义 Ansible 操作的目标主机清单。

**当前配置**：

```yaml
all:
  hosts:
    localhost:
      ansible_connection: local
```

**字段说明**：

- `all`：所有主机的顶级组
- `hosts`：主机列表
- `localhost`：本地主机（用于在当前机器上执行 Ansible 任务）
- `ansible_connection: local`：使用本地连接（不通过 SSH）

**适用场景**：

- 本项目仅操作本地机器（同步配置到 `~/.claude/`）
- 如需操作远程主机，可添加更多主机配置

---

### 2. `group_vars/all/settings.yml`

**用途**：存放公开配置变量，用于渲染 `claude-assets/settings.yml.j2` 模板。

**当前配置**：

```yaml
settings:
  env:
    ANTHROPIC_BASE_URL: https://openai.qiniu.com
    ANTHROPIC_DEFAULT_OPUS_MODEL: "claude-4.6-opus"
    ANTHROPIC_DEFAULT_SONNET_MODEL: "claude-4.5-sonnet"
    ANTHROPIC_DEFAULT_HAIKU_MODEL: "claude-4.6-haiku"
    CLAUDE_CODE_SUBAGENT_MODEL: "claude-4.5-sonnet"
  # outputStyle: "nekomata-engineer"
  outputStyle: "laowang-engineer"
```

**配置项说明**：

| 配置项 | 说明 | 示例值 |
| -------- | ------ | -------- |
| `ANTHROPIC_BASE_URL` | Anthropic API 的 Base URL | `https://openai.qiniu.com` |
| `ANTHROPIC_DEFAULT_OPUS_MODEL` | Opus 模型名称（用于计划模式） | `claude-4.6-opus` |
| `ANTHROPIC_DEFAULT_SONNET_MODEL` | Sonnet 模型名称（默认模型） | `claude-4.5-sonnet` |
| `ANTHROPIC_DEFAULT_HAIKU_MODEL` | Haiku 模型名称（后台功能） | `claude-4.6-haiku` |
| `CLAUDE_CODE_SUBAGENT_MODEL` | 子代理模型名称 | `claude-4.5-sonnet` |
| `outputStyle` | 输出风格名称（不含 .md 后缀） | `laowang-engineer` |

**注意事项**：

- 此文件可提交到 Git（不包含敏感信息）
- 模型名称应与 Anthropic API 支持的模型一致
- 输出风格名称应与 `claude-assets/output-styles/` 中的文件名匹配

---

### 3. `group_vars/all/secrets.yml`

**用途**：存放敏感配置变量（API Key、密码等），不应提交到版本控制。

**配置结构**（示例）：

```yaml
secrets:
  settings:
    env:
      ANTHROPIC_API_KEY: "your-api-key-here"
```

**安全建议**：

1. **添加到 .gitignore**：

   ```bash
   echo "inventory/*/group_vars/all/secrets.yml" >> .gitignore
   ```

2. **使用 Ansible Vault 加密**（可选）：

   ```bash
   ansible-vault encrypt inventory/default/group_vars/all/secrets.yml
   ```

3. **使用环境变量**（可选）：

   ```yaml
   secrets:
     settings:
       env:
         ANTHROPIC_API_KEY: "{{ lookup('env', 'ANTHROPIC_API_KEY') }}"
   ```

**注意事项**：

- 此文件包含敏感信息，切勿提交到公开仓库
- 如使用 Ansible Vault，需在运行 Playbook 时提供 Vault 密码

---

## 使用方法

### 修改公开配置

**步骤 1**：编辑 `settings.yml`

```bash
vim inventory/default/group_vars/all/settings.yml
```

**步骤 2**：修改配置项

```yaml
settings:
  env:
    ANTHROPIC_BASE_URL: https://api.anthropic.com  # 修改为官方 API
  outputStyle: "nekomata-engineer"                 # 切换输出风格
```

**步骤 3**：同步配置

```bash
ansible-playbook playbooks/sync_claude_config.yml
```

**步骤 4**：验证配置

```bash
cat ~/.claude/settings.json | jq .env.ANTHROPIC_BASE_URL
cat ~/.claude/settings.json | jq .outputStyle
```

---

### 修改敏感配置

**步骤 1**：编辑 `secrets.yml`

```bash
vim inventory/default/group_vars/all/secrets.yml
```

**步骤 2**：修改 API Key

```yaml
secrets:
  settings:
    env:
      ANTHROPIC_API_KEY: "sk-ant-xxxxx"  # 替换为你的 API Key
```

**步骤 3**：同步配置

```bash
ansible-playbook playbooks/sync_claude_config.yml
```

**步骤 4**：验证配置

```bash
# 检查 API Key 是否正确写入（注意安全）
cat ~/.claude/settings.json | jq .env.ANTHROPIC_API_KEY
```

---

### 添加新的环境

**步骤 1**：创建新环境目录

```bash
mkdir -p inventory/prod/group_vars/all
```

**步骤 2**：复制配置文件

```bash
cp inventory/default/inventory.yml inventory/prod/
cp inventory/default/group_vars/all/settings.yml inventory/prod/group_vars/all/
cp inventory/default/group_vars/all/secrets.yml inventory/prod/group_vars/all/
```

**步骤 3**：修改新环境的配置

```bash
vim inventory/prod/group_vars/all/settings.yml
vim inventory/prod/group_vars/all/secrets.yml
```

**步骤 4**：使用新环境运行 Playbook

```bash
ansible-playbook -i inventory/prod/inventory.yml playbooks/sync_claude_config.yml
```

---

## 开发指南

### 变量命名规范

**层级结构**：

```yaml
settings:              # 顶级命名空间（公开配置）
  env:                 # 环境变量
    KEY: "value"
  configName: "value"  # 其他配置

secrets:               # 顶级命名空间（敏感配置）
  settings:            # 与 settings 对应
    env:
      SENSITIVE_KEY: "value"
```

**命名约定**：

- 使用 `snake_case` 命名变量
- 环境变量使用 `UPPER_CASE`
- 嵌套层级不超过 3 层

---

### 变量优先级

Ansible 变量的优先级（从低到高）：

1. `inventory.yml` 中的主机变量
2. `group_vars/all/*.yml`（应用于所有主机）
3. `group_vars/<group_name>/*.yml`（应用于特定组）
4. `host_vars/<host_name>/*.yml`（应用于特定主机）
5. Playbook 中的 `vars`
6. 命令行 `-e` 参数

**最佳实践**：

- 通用配置放在 `group_vars/all/`
- 特定主机配置放在 `host_vars/`
- 敏感信息独立存储在 `secrets.yml`

---

### 变量引用

**在 Jinja2 模板中引用**：

```jinja2
# claude-assets/settings.yml.j2
env:
  ANTHROPIC_API_KEY: "{{ secrets.settings.env.ANTHROPIC_API_KEY }}"
  ANTHROPIC_BASE_URL: "{{ settings.env.ANTHROPIC_BASE_URL }}"
```

**在 Playbook 中引用**：

```yaml
- name: 打印配置
  debug:
    msg: "Base URL: {{ settings.env.ANTHROPIC_BASE_URL }}"
```

---

### 添加新的配置项

**步骤 1**：在 `settings.yml` 中添加变量

```yaml
settings:
  myNewConfig:
    key1: "value1"
    key2: "value2"
```

**步骤 2**：在模板中引用变量

```jinja2
# claude-assets/settings.yml.j2
myNewConfig:
  key1: "{{ settings.myNewConfig.key1 }}"
  key2: "{{ settings.myNewConfig.key2 }}"
```

**步骤 3**：同步配置并验证

```bash
ansible-playbook playbooks/sync_claude_config.yml
cat ~/.claude/settings.json | jq .myNewConfig
```

---

## 常见问题

### Q1：API Key 未生效？

**原因**：可能未正确配置 `secrets.yml` 或未同步

**解决方法**：

```bash
# 1. 检查 secrets.yml 是否存在且配置正确
cat inventory/default/group_vars/all/secrets.yml

# 2. 重新同步配置
ansible-playbook playbooks/sync_claude_config.yml

# 3. 验证 settings.json 中的 API Key
cat ~/.claude/settings.json | jq .env.ANTHROPIC_API_KEY
```

---

### Q2：配置修改后未生效？

**原因**：未运行同步 Playbook

**解决方法**：

每次修改 `settings.yml` 或 `secrets.yml` 后，必须运行：

```bash
ansible-playbook playbooks/sync_claude_config.yml
```

---

### Q3：如何备份配置？

**方法 1**：备份整个 inventory 目录

```bash
tar -czf inventory-backup-$(date +%Y%m%d).tar.gz inventory/
```

**方法 2**：提交到私有 Git 仓库

```bash
# 确保 secrets.yml 已加密或使用私有仓库
git add inventory/
git commit -m "backup: save inventory configuration"
git push
```

---

### Q4：如何共享配置（不泄露 API Key）？

**方法 1**：使用 Ansible Vault

```bash
# 加密 secrets.yml
ansible-vault encrypt inventory/default/group_vars/all/secrets.yml

# 共享加密后的文件和密码（安全渠道）
```

**方法 2**：使用环境变量

```yaml
# secrets.yml
secrets:
  settings:
    env:
      ANTHROPIC_API_KEY: "{{ lookup('env', 'ANTHROPIC_API_KEY') }}"
```

```bash
# 设置环境变量
export ANTHROPIC_API_KEY="sk-ant-xxxxx"

# 运行 Playbook
ansible-playbook playbooks/sync_claude_config.yml
```

---

### Q5：如何切换环境？

**方法 1**：使用 `-i` 参数指定 inventory

```bash
ansible-playbook -i inventory/prod/inventory.yml playbooks/sync_claude_config.yml
```

**方法 2**：修改 `ansible.cfg` 中的默认 inventory

```ini
# ansible.cfg
[defaults]
inventory = $PWD/inventory/prod/inventory.yml
```

---

**最后更新时间**: 2026-02-17T05:32:00+00:00
