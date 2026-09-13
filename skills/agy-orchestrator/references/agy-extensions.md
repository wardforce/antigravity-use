# AGY 扩展发现：MCP、Skills、Plugins、Hooks、Rules

## 使用原则

先发现并复用已存在的扩展，再决定 AGY 步骤合同。读取配置不等于扩展已成功加载；
用 CLI 状态、实际工具列表或执行结果确认运行态。安装、卸载、启停、修改配置、扩大
权限或连接新的外部服务属于额外状态变更，只有用户明确要求时才执行。

配置中可能含 `env`、HTTP `headers`、OAuth 客户端信息或 token。向 AGY 提供完成步骤
所需的最小信息，并在报告中隐藏凭据值。

## 路径速查

| 组件 | 项目级 | 通用 Antigravity 全局级 | AGY CLI 专用或暂存位置 |
|---|---|---|---|
| MCP | `<workspace>/.agents/mcp_config.json` | `~/.gemini/config/mcp_config.json` | CLI 使用相同的项目级和全局 MCP 配置 |
| Skills | `<workspace>/.agents/skills/` | `~/.gemini/config/skills/<skill-folder>/` | `~/.gemini/antigravity-cli/skills/` |
| Plugins | `<workspace>/.agents/plugins/` 或 `<workspace>/_agents/plugins/` | `~/.gemini/config/plugins/` | `~/.gemini/antigravity-cli/plugins/<plugin_name>/` |
| Hooks | `<workspace>/.agents/hooks.json` | `~/.gemini/config/hooks.json` | 插件根目录 `hooks.json`；CLI 也可由主 `settings.json` 配置 |
| Rules | `<workspace>/.agents/rules/` | `~/.gemini/GEMINI.md` | 插件根目录 `rules/` |

Skills 的项目入口在各产品中都位于 `.agents/skills/`，全局入口不同：通用 Antigravity
使用 `~/.gemini/config/skills/`，AGY CLI 使用
`~/.gemini/antigravity-cli/skills/`。

官方 Skills 页面采用开放标准目录形式：

```text
.agents/skills/<skill-folder>/SKILL.md
~/.gemini/config/skills/<skill-folder>/SKILL.md
```

CLI 的 Plugins & Skills 页面还记录了扁平 Markdown 形式：

```text
.agents/skills/<skill-name>.md
~/.gemini/antigravity-cli/skills/<skill-name>.md
```

因此，共享项目入口相同，但全局目录和文档化的包装形式不同。读取时兼容两种项目布局；
为通用开放标准创建项目 Skill 时优先使用 `<skill-folder>/SKILL.md`。为 AGY CLI 创建
全局 slash command 时按 CLI 页面使用其专用目录和 Markdown 文件。

## MCP

MCP 为 AGY 提供本地工具、数据库、文件解析器和远程 API。先用只读命令检查运行态：

```powershell
agy mcp list
```

交互式 TUI 可使用 `/mcp` 查看状态、重载配置和连接日志。CLI 还提供 `agy mcp add`、
`remove`、`enable`、`disable`；这些命令会改变配置，仅在用户明确要求时使用。

配置根对象是 `mcpServers`：

```json
{
  "mcpServers": {
    "local-server": {
      "command": "node",
      "args": ["/path/to/server.js"],
      "env": {"SERVICE_TOKEN": "REDACTED"},
      "cwd": "/path/to/workdir"
    },
    "remote-server": {
      "serverUrl": "https://api.example.com/mcp/",
      "headers": {"Authorization": "Bearer REDACTED"}
    }
  }
}
```

传输必须提供 `command` 或 `serverUrl`。可选字段包括 `args`、`env`、`cwd`、`headers`、
`authProviderType`、`oauth`、`disabled`、`disabledTools`。远程 SSE、Streamable HTTP 或
WebSocket 配置使用 `serverUrl`；官方页面明确不采用旧字段 `url` 或 `httpUrl`。

未配置的 MCP 工具默认进入 Ask。权限匹配形式为：

```text
mcp(server/tool)
mcp(server/*)
mcp(*)
```

编排时只把已连接、未禁用且步骤所需的工具写入合同。服务器存在于配置文件但状态未
确认时，将其标为“已配置，运行态待验证”。

## Skills

Skill 是带 `SKILL.md` 的开放标准知识包，可包含 `scripts/`、`examples/` 和
`resources/`。AGY 在会话开始时先看到名称和 description，匹配后再读取正文，属于
渐进式披露。

发现顺序：

1. 检查项目 `.agents/skills/` 中与当前任务相关的 folder/`SKILL.md` 和 CLI 扁平 `.md`。
2. 仅在任务确实需要跨项目能力且权限允许时，检查对应产品的全局 Skills 目录。
3. 记录 Skill 来源，避免同名项目级、全局级或插件 Skill 被重复注入。
4. 如果 Skill 引用脚本，先运行脚本的 `--help`，避免把整段源码无条件加载进上下文。

AGY CLI 会把已注册的 Markdown Skill 转为 TUI slash command，例如 `/format-tests`。

## Plugins

Plugin 是带命名空间的组合包，可同时提供 Skills、Rules、MCP、Hooks；AGY CLI 插件还
可以包含 `agents/`：

```text
<plugin>/
├── plugin.json
├── mcp_config.json
├── hooks.json
├── skills/
├── agents/
└── rules/
```

`plugin.json` 是必需标记文件。推荐使用官方 schema：

```json
{
  "$schema": "https://antigravity.google/schemas/v1/plugin.json",
  "name": "my-plugin",
  "description": "Plugin purpose"
}
```

`name` 只使用字母、数字、连字符和下划线。只读检查使用：

```powershell
agy plugin list
```

状态变更命令包括：

```powershell
agy plugin install <path-or-target>
agy plugin disable <plugin_name>
agy plugin enable <plugin_name>
agy plugin uninstall <plugin_name>
```

已加载插件提供同名组件时，记录它们来自该插件，不再复制一份独立 MCP、Hook 或 Rule
配置。插件安装目录与源目录含义不同：CLI 会把已安装或导入的包暂存到
`~/.gemini/antigravity-cli/plugins/`。

## Hooks

Hook 在模型调用或工具调用前后执行命令，可改变权限、注入步骤、继续执行或终止流程。
开始 AGY 步骤前读取适用的项目、全局和插件 `hooks.json`，并在验收时把 Hook 生成的
文件、拒绝和输出纳入 diff 与日志检查。

支持事件：

| 事件 | 作用 |
|---|---|
| `PreToolUse` | 工具执行前；matcher 匹配工具名 |
| `PostToolUse` | 工具完成后；matcher 匹配工具名 |
| `PreInvocation` | 模型调用前；matcher 被忽略 |
| `PostInvocation` | 模型调用后；matcher 被忽略 |
| `Stop` | 执行循环准备结束时 |

`PreToolUse`/`PostToolUse` matcher 是正则表达式，例如 `run_command`、
`run_command|view_file`、`browser_.*` 或 `*`。Handler 当前类型为 `command`，默认超时
30 秒；`enabled: false` 可关闭单个 Hook。

Hook 从 stdin 接收 camelCase JSON，从 stdout 返回 JSON。公共输入包含
`conversationId`、`workspacePaths`、`transcriptPath`、`artifactDirectoryPath` 和
`modelName`。`PreToolUse` 的 decision 可以是 `allow`、`deny`、`ask`、`force_ask` 或
`deny_unless_prior_grant`。Hook 导致的权限拒绝可能表现为 AGY JSON 中的
`denied_actions`，仍按步骤失败处理。

TUI 中使用 `/hooks` 检查已加载 Hook。查看不等于修改；更新 `hooks.json` 或主
`settings.json` 需要用户明确要求。

## Rules

Rules 是 Agent 遵循的 Markdown 约束，每个文件最多 12,000 字符。项目规则位于
`.agents/rules/`，旧 `.agent/rules/` 仍受兼容；全局规则位于 `~/.gemini/GEMINI.md`。

四种激活方式：

1. `Manual`：用户通过 @mention 手动激活。
2. `Always On`：始终应用。
3. `Model Decision`：模型根据自然语言 description 决定。
4. `Glob`：按 `*.js`、`src/**/*.ts` 等文件模式应用。

Rule 可用 `@filename` 引用其他文件。相对路径相对 Rule 文件解析；绝对路径先按真实
绝对路径解析，目标不存在时再尝试仓库相对路径。编排者应把当前步骤实际激活的规则
写入合同，而不是把发现到的全部规则都塞进上下文。

## 发现与委派顺序

1. 读取项目级配置并确定与任务相关的扩展候选。
2. 需要跨项目能力时再读取对应产品的全局目录。
3. 使用 `agy mcp list`、`agy plugin list`、TUI `/mcp` 或 `/hooks` 核对运行态。
4. 把已确认扩展、适用 Rule、Hook 副作用和权限要求写入步骤合同。
5. AGY 返回后检查扩展实际调用、Hook 输出、`denied_actions`、产物绝对路径和真实 diff。

## 官方来源

- CLI MCP：https://antigravity.google/docs/cli/mcp/
- 通用 MCP：https://antigravity.google/docs/mcp/
- Skills：https://antigravity.google/docs/skills/
- Plugins：https://antigravity.google/docs/plugins/
- CLI Plugins & Skills：https://antigravity.google/docs/cli/plugins/
- Hooks：https://antigravity.google/docs/hooks/
- Rules：https://antigravity.google/docs/rules-workflows/

