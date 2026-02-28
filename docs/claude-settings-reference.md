# Claude Code Settings.json 完整字段参考

> 基于 JSON Schema: https://json.schemastore.org/claude-code-settings.json
> 更新日期: 2026-02-28

## 核心配置字段

### 模型配置

| 字段 | 类型 | 默认值 | 描述 |
|------|------|--------|------|
| `model` | string | N/A | 覆盖默认模型（如 "opus[1m]"、"sonnet"、"claude-4.6-opus"） |
| `availableModels` | array | N/A | 限制用户可选模型列表（多层级合并去重） |
| `effortLevel` | string | "high" | Opus 4.6 推理努力级别（low/medium/high） |
| `fastMode` | boolean | false | 启用 Opus 4.6 快速模式（2.5x 输出速度，更高成本） |
| `alwaysThinkingEnabled` | boolean | N/A | 默认启用扩展思考模式（Thinking Mode） |

**相关文档**: [Model Config](https://code.claude.com/docs/en/model-config)

---

### 环境变量

| 字段 | 类型 | 默认值 | 描述 |
|------|------|--------|------|
| `env` | object | {} | 环境变量配置（支持所有 Claude Code 环境变量） |

**常用环境变量**:
- `ANTHROPIC_API_KEY` - API 密钥
- `ANTHROPIC_BASE_URL` - API 基础 URL（如使用代理）
- `ANTHROPIC_DEFAULT_OPUS_MODEL` - Opus 模型名称
- `ANTHROPIC_DEFAULT_SONNET_MODEL` - Sonnet 模型名称
- `ANTHROPIC_DEFAULT_HAIKU_MODEL` - Haiku 模型名称
- `CLAUDE_CODE_SUBAGENT_MODEL` - 子代理使用的模型
- `CLAUDE_CODE_DISABLE_1M_CONTEXT` - 禁用 1M token 上下文窗口
- `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` - 禁用非必要网络流量
- `DISABLE_TELEMETRY` - 禁用遥测
- `DISABLE_ERROR_REPORTING` - 禁用错误报告
- `MCP_TIMEOUT` - MCP 服务器超时时间（毫秒）

---

### 权限配置

| 字段 | 类型 | 默认值 | 描述 |
|------|------|--------|------|
| `permissions` | object | N/A | 工具使用权限配置 |
| `permissions.allow` | array | N/A | 允许的权限规则列表 |
| `permissions.ask` | array | N/A | 需要确认的权限规则列表 |
| `permissions.deny` | array | N/A | 拒绝的权限规则列表 |
| `permissions.defaultMode` | string | N/A | 默认权限模式（default/acceptEdits/plan/dontAsk/bypassPermissions/delegate） |
| `permissions.additionalDirectories` | array | N/A | 额外允许访问的目录 |
| `permissions.disableBypassPermissionsMode` | string | N/A | 禁用绕过权限模式（值为 "disable"） |

**权限规则语法示例**:
```json
{
  "allow": [
    "Bash",
    "Bash(git *)",
    "Edit(*.ts)",
    "Read(.env)",
    "WebFetch(domain:example.com)"
  ],
  "deny": [
    "Bash(rm -rf *)",
    "Write(~/.ssh/**)"
  ]
}
```

**相关文档**: [Permissions](https://code.claude.com/docs/en/permissions)

---

### 插件与 MCP 服务器

| 字段 | 类型 | 默认值 | 描述 |
|------|------|--------|------|
| `enabledPlugins` | object | N/A | 启用的插件（插件ID@市场ID: true） |
| `pluginConfigs` | object | N/A | 每个插件的配置（按插件ID键入） |
| `enableAllProjectMcpServers` | boolean | N/A | 自动批准项目中所有 MCP 服务器 |
| `enabledMcpjsonServers` | array | N/A | 从 .mcp.json 批准的 MCP 服务器列表 |
| `disabledMcpjsonServers` | array | N/A | 从 .mcp.json 拒绝的 MCP 服务器列表 |
| `allowedMcpServers` | array | N/A | 企业白名单（允许使用的 MCP 服务器） |
| `deniedMcpServers` | array | N/A | 企业黑名单（明确禁止的 MCP 服务器） |
| `allowManagedMcpServersOnly` | boolean | N/A | 仅允许托管设置中的 MCP 服务器 |

**相关文档**: [Plugins](https://code.claude.com/docs/en/plugins), [MCP](https://code.claude.com/docs/en/mcp)

---

### Hooks（钩子）

| 字段 | 类型 | 默认值 | 描述 |
|------|------|--------|------|
| `hooks` | object | N/A | 自定义命令钩子（在工具执行前后运行） |
| `hooks.SessionStart` | array | N/A | 会话启动时运行 |
| `hooks.SessionEnd` | array | N/A | 会话结束时运行 |
| `hooks.PreToolUse` | array | N/A | 工具执行前运行 |
| `hooks.PostToolUse` | array | N/A | 工具执行成功后运行 |
| `hooks.PostToolUseFailure` | array | N/A | 工具执行失败后运行 |
| `hooks.UserPromptSubmit` | array | N/A | 用户提交提示时运行 |
| `hooks.Stop` | array | N/A | 代理响应结束时运行 |
| `hooks.PermissionRequest` | array | N/A | 权限对话框出现时运行 |
| `hooks.TaskCompleted` | array | N/A | 任务标记为完成时运行 |
| `hooks.ConfigChange` | array | N/A | 设置文件变更时运行 |
| `hooks.Notification` | array | N/A | 通知触发时运行 |
| `hooks.PreCompact` | array | N/A | 上下文压缩前运行 |
| `hooks.SubagentStart` | array | N/A | 子代理启动时运行 |
| `hooks.SubagentStop` | array | N/A | 子代理结束时运行 |
| `hooks.TeammateIdle` | array | N/A | 团队成员空闲时运行 |
| `hooks.WorktreeCreate` | array | N/A | 创建工作树时运行 |
| `hooks.WorktreeRemove` | array | N/A | 移除工作树时运行 |
| `hooks.Setup` | array | N/A | 仓库初始化时运行（未文档化） |
| `disableAllHooks` | boolean | N/A | 禁用所有钩子（包括 statusLine） |
| `allowManagedHooksOnly` | boolean | N/A | 仅允许托管钩子（企业功能） |

**Hook 命令格式**:
```json
{
  "hooks": {
    "SessionStart": [
      {
        "type": "command",
        "command": "echo 'Session started'",
        "timeout": 30,
        "async": true
      }
    ]
  }
}
```

**相关文档**: [Hooks](https://code.claude.com/docs/en/hooks)

---

### 输出风格与 UI

| 字段 | 类型 | 默认值 | 描述 |
|------|------|--------|------|
| `outputStyle` | string | N/A | 输出风格名称（引用 ~/.claude/output-styles/*.md） |
| `language` | string | N/A | 首选响应语言 |
| `showTurnDuration` | boolean | true | 显示回合耗时消息（如 "Cooked for 1m 6s"） |
| `spinnerTipsEnabled` | boolean | true | 显示 Spinner 提示 |
| `spinnerTipsOverride` | object | N/A | 自定义 Spinner 提示 |
| `spinnerVerbs` | object | N/A | 自定义 Spinner 进度文本动词 |
| `terminalProgressBarEnabled` | boolean | true | 启用终端进度条（Windows Terminal/iTerm2） |
| `prefersReducedMotion` | boolean | false | 减少或禁用 UI 动画（无障碍） |

**SpinnerTipsOverride 示例**:
```json
{
  "spinnerTipsOverride": {
    "tips": ["自定义提示 1", "自定义提示 2"],
    "excludeDefault": false
  }
}
```

---

### StatusLine（状态行）

| 字段 | 类型 | 默认值 | 描述 |
|------|------|--------|------|
| `statusLine` | object | N/A | 自定义状态行显示配置 |
| `statusLine.type` | string | N/A | 状态行类型（必须为 "command"） |
| `statusLine.command` | string | N/A | Shell 命令或脚本路径（从 stdin 读 JSON，输出到 stdout） |
| `statusLine.padding` | number | 0 | 额外水平间距字符数 |

**相关文档**: [StatusLine](https://code.claude.com/docs/en/statusline)

---

### Git 归属配置

| 字段 | 类型 | 默认值 | 描述 |
|------|------|--------|------|
| `attribution` | object | N/A | 自定义 Git 提交和 PR 归属 |
| `attribution.commit` | string | N/A | Git 提交归属（包括 trailers）。空字符串隐藏归属 |
| `attribution.pr` | string | N/A | PR 描述归属。空字符串隐藏归属 |
| `includeCoAuthoredBy` | boolean | true | **已废弃**，使用 `attribution` 替代 |

**相关文档**: [Attribution Settings](https://code.claude.com/docs/en/settings#attribution-settings)

---

### 沙箱配置

| 字段 | 类型 | 默认值 | 描述 |
|------|------|--------|------|
| `sandbox` | object | N/A | 沙箱执行配置 |
| `sandbox.enabled` | boolean | N/A | 启用沙箱 Bash |
| `sandbox.autoAllowBashIfSandboxed` | boolean | true | 沙箱命令自动允许（不提示） |
| `sandbox.allowUnsandboxedCommands` | boolean | N/A | 允许通过 dangerouslyDisableSandbox 跳过沙箱 |
| `sandbox.excludedCommands` | array | N/A | 不进沙箱的命令列表（如 ["git", "docker"]） |
| `sandbox.ignoreViolations` | object | N/A | 忽略违规的命令模式到路径映射 |
| `sandbox.enableWeakerNestedSandbox` | boolean | N/A | 启用较弱嵌套沙箱（无特权 Docker 环境） |
| `sandbox.network` | object | N/A | 网络隔离设置 |
| `sandbox.network.allowedDomains` | array | N/A | 白名单域（支持 *.example.com） |
| `sandbox.network.allowManagedDomainsOnly` | boolean | N/A | 仅允许托管设置中的域 |
| `sandbox.network.allowLocalBinding` | boolean | N/A | 允许绑定本地网络地址（localhost 端口） |
| `sandbox.network.allowUnixSockets` | array | N/A | 允许 Unix 套接字路径列表 |
| `sandbox.network.allowAllUnixSockets` | boolean | N/A | 允许所有 Unix 套接字 |
| `sandbox.network.httpProxyPort` | integer | N/A | HTTP 代理端口 |
| `sandbox.network.socksProxyPort` | integer | N/A | SOCKS 代理端口 |

**相关文档**: [Sandboxing](https://code.claude.com/docs/en/sandboxing)

---

### 插件市场管理

| 字段 | 类型 | 默认值 | 描述 |
|------|------|--------|------|
| `extraKnownMarketplaces` | object | N/A | 额外插件市场（确保团队成员访问） |
| `strictKnownMarketplaces` | array | N/A | 白名单市场（未定义=无限制，空数组=锁定） |
| `blockedMarketplaces` | array | N/A | 黑名单市场（下载前阻止） |
| `skippedMarketplaces` | array | N/A | 用户选择不安装的市场 |
| `skippedPlugins` | array | N/A | 用户选择不安装的插件 |

**相关文档**: [Plugin Marketplaces](https://code.claude.com/docs/en/plugin-marketplaces)

---

### 会话与存储

| 字段 | 类型 | 默认值 | 描述 |
|------|------|--------|------|
| `cleanupPeriodDays` | integer | 30 | 保留聊天记录天数（0 禁用清理） |
| `plansDirectory` | string | "~/.claude/plans" | 计划文件存储目录 |
| `respectGitignore` | boolean | true | @ 文件选择器是否遵守 .gitignore |

---

### 认证与企业功能

| 字段 | 类型 | 默认值 | 描述 |
|------|------|--------|------|
| `apiKeyHelper` | string | N/A | 输出认证值的脚本路径 |
| `awsCredentialExport` | string | N/A | 导出 AWS 凭证的脚本 |
| `awsAuthRefresh` | string | N/A | 刷新 AWS 认证的脚本 |
| `forceLoginMethod` | string | N/A | 强制登录方式（claudeai/console） |
| `forceLoginOrgUUID` | string | N/A | OAuth 登录组织 UUID |
| `otelHeadersHelper` | string | N/A | 输出 OpenTelemetry 头的脚本 |
| `skipWebFetchPreflight` | boolean | N/A | 跳过 WebFetch 黑名单检查（企业限制环境） |

---

### 托管设置（企业）

| 字段 | 类型 | 默认值 | 描述 |
|------|------|--------|------|
| `allowManagedHooksOnly` | boolean | N/A | 仅允许托管钩子（阻止用户/项目钩子） |
| `allowManagedPermissionRulesOnly` | boolean | N/A | 仅允许托管权限规则 |
| `allowManagedMcpServersOnly` | boolean | N/A | 仅允许托管 MCP 服务器白名单 |

**相关文档**: [Managed Settings](https://code.claude.com/docs/en/settings)

---

### 更新与公告

| 字段 | 类型 | 默认值 | 描述 |
|------|------|--------|------|
| `autoUpdatesChannel` | string | "latest" | 更新频道（stable/latest） |
| `companyAnnouncements` | array | N/A | 启动时显示的公司公告（随机选择一条） |

---

### 文件建议

| 字段 | 类型 | 默认值 | 描述 |
|------|------|--------|------|
| `fileSuggestion` | object | N/A | 自定义 @ 文件自动完成脚本 |
| `fileSuggestion.type` | string | N/A | 类型（必须为 "command"） |
| `fileSuggestion.command` | string | N/A | Shell 命令 |

**相关文档**: [File Suggestion Settings](https://code.claude.com/docs/en/settings#file-suggestion-settings)

---

### 团队协作（实验性）

| 字段 | 类型 | 默认值 | 描述 |
|------|------|--------|------|
| `teammateMode` | string | "auto" | 团队成员显示模式（auto/in-process/tmux） |

**启用方式**: 需在 `env` 中添加 `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS`

**相关文档**: [Agent Teams](https://code.claude.com/docs/en/agent-teams)

---

## 命令行覆盖选项

以下命令行参数可覆盖 settings.json 配置：

| 参数 | 对应字段 | 描述 |
|------|---------|------|
| `--model <model>` | `model` | 覆盖模型 |
| `--agent <agent>` | N/A | 指定 Agent |
| `--agents <json>` | N/A | 自定义 Agent 定义 |
| `--effort <level>` | `effortLevel` | 努力级别（low/medium/high） |
| `--output-style <style>` | `outputStyle` | 输出风格 |
| `--permission-mode <mode>` | `permissions.defaultMode` | 权限模式 |
| `--allowed-tools <tools>` | `permissions.allow` | 允许的工具 |
| `--disallowed-tools <tools>` | `permissions.deny` | 禁止的工具 |
| `--add-dir <dirs>` | `permissions.additionalDirectories` | 额外目录 |
| `--settings <file-or-json>` | N/A | 加载额外设置 |
| `--mcp-config <configs>` | N/A | 加载 MCP 服务器 |
| `--env <key=value>` | `env` | 设置环境变量 |
| `--dangerously-skip-permissions` | N/A | 绕过权限检查 |
| `--allow-dangerously-skip-permissions` | N/A | 启用绕过选项 |
| `--chrome` / `--no-chrome` | N/A | Chrome 集成 |
| `--ide` | N/A | 自动连接 IDE |
| `--debug [filter]` | N/A | 调试模式 |
| `--debug-file <path>` | N/A | 调试日志文件 |

---

## Schema 元数据

| 字段 | 类型 | 默认值 | 描述 |
|------|------|--------|------|
| `$schema` | string | N/A | JSON Schema 引用 |

**标准值**: `"https://json.schemastore.org/claude-code-settings.json"`

---

## 工具列表

### 可用于权限规则的工具

- `Bash` - 执行 Shell 命令
- `Edit` - 编辑文件
- `MultiEdit` - 批量编辑多个文件
- `Read` - 读取文件
- `Write` - 写入文件
- `Glob` - 文件模式匹配
- `Grep` - 内容搜索
- `NotebookEdit` - 编辑 Jupyter Notebook
- `NotebookRead` - 读取 Jupyter Notebook
- `LSP` - 语言服务器协议
- `Task` - 任务管理
- `TaskCreate` - 创建任务
- `TaskGet` - 获取任务
- `TaskList` - 列出任务
- `TaskUpdate` - 更新任务
- `TaskOutput` - 任务输出
- `TaskStop` - 停止任务
- `TodoWrite` - 写入 TODO
- `WebFetch` - 获取网页内容
- `WebSearch` - 网页搜索
- `Skill` - 自定义技能
- `ToolSearch` - 工具搜索
- `ExitPlanMode` - 退出计划模式
- `KillShell` - 终止 Shell
- `LS` - 列出目录（已废弃，使用 Glob）
- `mcp__*` - MCP 工具（使用 `mcp__<server>__<tool>` 格式）

**相关文档**: [Tools Available](https://code.claude.com/docs/en/settings#tools-available-to-claude)

---

## 设置层级

Claude Code 支持多层级设置（优先级从高到低）：

1. **命令行参数** - `--model`, `--effort` 等
2. **Local Settings** - `./.claude/local-settings.json`（Git 忽略）
3. **Project Settings** - `./.claude/settings.json`（Git 跟踪）
4. **User Settings** - `~/.claude/settings.json`（全局配置）
5. **Managed Settings** - 企业托管设置（最低优先级）

**特殊合并规则**:
- 数组字段（如 `availableModels`）: 去重合并
- 权限规则（`permissions.allow/ask/deny`）: 累加合并
- 对象字段（如 `env`）: 深度合并
- 标量字段（如 `model`）: 高优先级覆盖

---

## 示例配置

### 基础配置

```json
{
  "$schema": "https://json.schemastore.org/claude-code-settings.json",
  "model": "opus[1m]",
  "effortLevel": "high",
  "alwaysThinkingEnabled": true,
  "outputStyle": "engineer-professional",
  "permissions": {
    "defaultMode": "acceptEdits",
    "allow": ["Bash", "Edit", "Read", "Write", "WebSearch"]
  }
}
```

### 企业配置

```json
{
  "$schema": "https://json.schemastore.org/claude-code-settings.json",
  "allowManagedPermissionRulesOnly": true,
  "allowManagedMcpServersOnly": true,
  "strictKnownMarketplaces": ["https://plugins.anthropic.com"],
  "permissions": {
    "allow": ["Read", "WebSearch"],
    "deny": ["Bash(rm*)", "Write(~/.ssh/**)"]
  },
  "sandbox": {
    "enabled": true,
    "network": {
      "allowedDomains": ["*.example.com", "api.company.com"]
    }
  }
}
```

### 完整配置（带环境变量和钩子）

```json
{
  "$schema": "https://json.schemastore.org/claude-code-settings.json",
  "model": "sonnet",
  "effortLevel": "medium",
  "fastMode": false,
  "alwaysThinkingEnabled": true,
  "outputStyle": "nekomata-engineer",
  "language": "zh-CN",
  "env": {
    "ANTHROPIC_API_KEY": "sk-xxx",
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "claude-4.6-opus",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "claude-4.5-sonnet",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "claude-4.6-haiku",
    "CLAUDE_CODE_SUBAGENT_MODEL": "claude-4.5-sonnet",
    "DISABLE_TELEMETRY": "1"
  },
  "permissions": {
    "defaultMode": "acceptEdits",
    "allow": [
      "Bash",
      "Edit",
      "Read",
      "Write",
      "WebSearch",
      "WebFetch"
    ],
    "deny": [
      "Bash(rm -rf *)",
      "Write(~/.ssh/**)"
    ]
  },
  "enabledPlugins": {
    "playwright@claude-plugins-official": true,
    "context7@claude-plugins-official": true,
    "github@claude-plugins-official": true
  },
  "hooks": {
    "SessionStart": [
      {
        "type": "command",
        "command": "echo 'Welcome to Claude Code!'",
        "async": true
      }
    ],
    "PostToolUse": [
      {
        "type": "command",
        "command": "echo 'Tool executed: $TOOL_NAME'",
        "matchers": ["Bash", "Edit"]
      }
    ]
  },
  "statusLine": {
    "type": "command",
    "command": "ccline",
    "padding": 1
  },
  "attribution": {
    "commit": "Co-authored-by: Claude <claude@anthropic.com>",
    "pr": "\n\n---\n*Generated with Claude Code*"
  },
  "showTurnDuration": true,
  "spinnerTipsEnabled": true,
  "terminalProgressBarEnabled": true,
  "cleanupPeriodDays": 30,
  "respectGitignore": true
}
```

---

## 参考链接

- [官方文档](https://code.claude.com/docs)
- [JSON Schema](https://json.schemastore.org/claude-code-settings.json)
- [Settings Guide](https://code.claude.com/docs/en/settings)
- [Model Config](https://code.claude.com/docs/en/model-config)
- [Permissions](https://code.claude.com/docs/en/permissions)
- [Hooks](https://code.claude.com/docs/en/hooks)
- [Plugins](https://code.claude.com/docs/en/plugins)
- [MCP](https://code.claude.com/docs/en/mcp)
- [Sandboxing](https://code.claude.com/docs/en/sandboxing)
- [Output Styles](https://code.claude.com/docs/en/output-styles)
