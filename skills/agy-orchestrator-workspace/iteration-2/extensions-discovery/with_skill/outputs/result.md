# AGY 执行前方案

## 当前边界

本轮用户明确要求使用 AGY，且任务被定义为多步骤项目任务；因此满足 `agy-orchestrator` 的触发门。但用户同时要求“只输出执行前方案”，本轮不启动 AGY 执行、不安装或启停扩展、不修改配置。唯一允许写入的文件是本文件。

工作区基线（只读检查）：当前已有修改和未跟踪内容属于用户工作，尤其是 `docs/superpowers/*`、`skills/agy-orchestrator/*`、`skills/agy-orchestrator-workspace/`；后续若获准执行，必须保留它们并只比较本次新增 diff。

## 主编排模型的扩展发现与复用顺序

1. 先读仓库指令和本技能引用的三份参考：模型路由、编排协议、AGY 扩展发现。
2. 从项目入口开始发现：检查 `E:\project\antigravity-use\.agents\mcp_config.json`、`.agents\skills\`（同时兼容扁平 `.md`）、`.agents\plugins\`、`_agents\plugins\`、`.agents\hooks.json`、`.agents\rules\` 及兼容旧目录 `.agent\rules\`。本次检查未发现项目级 `.agents` 内容，因此没有项目级扩展可以注入合同。
3. 仅在任务需要跨项目能力时读取通用 Antigravity 全局目录：MCP 的 `C:\Users\ward\.gemini\config\mcp_config.json`；Skills 的 `C:\Users\ward\.gemini\config\skills\`；Plugins 的 `C:\Users\ward\.gemini\config\plugins\`；Hooks 的 `C:\Users\ward\.gemini\config\hooks.json`；Rules 的 `C:\Users\ward\.gemini\GEMINI.md`。本次任务需要解释扩展发现，因此已做只读检查。
4. 用运行态而不是文件存在性确认是否可复用：运行 `agy mcp list`、`agy plugin list`；必要时用 TUI 的 `/mcp`、`/hooks` 查看连接和加载状态。只有“已连接、未禁用、且当前步骤需要”的 MCP/Skill/Plugin/Hook/Rule 才进入 AGY 步骤合同；配置中存在但状态未确认的项目标记为“已配置，运行态待验证”。
5. 每一步调用前运行 `agy models`，只使用当次返回的实时 slug；把最小成本模型写入合同。当前 `agy models` 返回“Please sign in to view available models”，所以实时 slug 尚未可用，不能猜测或静默使用快照名称。
6. 对插件已提供的同名 Skills、Rules、MCP、Hooks 去重，记录其插件来源；读取含 `env`、headers、OAuth 或 token 的配置时只记录名称、字段和状态，绝不把凭据值传给 AGY 或写入报告。

## 本机发现结果

- AGY CLI 可执行文件：`C:\Users\ward\AppData\Local\agy\bin\agy.exe`（版本日志为 1.2.2）。未发现名为 `antigravity` 的独立 PATH 命令。
- 项目级：`E:\project\antigravity-use\.agents\` 当前未发现，因此项目级 MCP/Skill/Plugin/Hook/Rule 均为“未发现”。
- 通用全局 MCP：`C:\Users\ward\.gemini\config\mcp_config.json` 中发现 `codegraph`，`agy mcp list` 显示 `stdio / enabled`；真正调用前仍按步骤需要性复核。
- 通用全局 Plugins：`C:\Users\ward\.gemini\config\plugins\` 中存在 `agent-skills`、`superpowers`、`ponytail`、`i-have-adhd`、`google-antigravity-sdk`、`chrome-devtools-plugin`、`modern-web-guidance-plugin`、`android-cli-plugin`、`flutter`、`science` 等目录；`agy plugin list` 报告已导入的 `ponytail`、`superpowers`、`agent-skills`、`i-have-adhd` 及其组件。插件目录存在不等于每个组件在当前会话已激活。
- 通用全局 Skills：`C:\Users\ward\.gemini\config\skills\` 存在开放标准 Skill 目录；项目没有同名 Skill 可覆盖。AGY CLI 专用 Skills 目录当前未发现可列出的内容。
- 通用全局 Rules：`C:\Users\ward\.gemini\GEMINI.md` 存在，并包含本项目的 ADHD 友好输出约束；插件根目录下的 `rules/` 只在对应插件实际加载时纳入合同。
- Hooks：`C:\Users\ward\.gemini\config\hooks.json` 存在。运行日志报告从 1 个 hooks 文件加载 2 个 named hooks；配置包含 Bash 的 `PreToolUse`（`dcg`）以及 `orca-status` 的 `PreInvocation`、`PostInvocation`、`Stop`、全工具 `PostToolUse`。Hook 的拒绝或生成物必须计入 AGY 验收。
- 运行态限制：本次 `agy models` 因未登录失败；日志还显示 CLI 日志/崩溃目录存在访问权限告警。它们不是本轮配置修复目标。

## AGY CLI 与通用 Antigravity 的精确路径差异

| 组件 | 项目级（两者共享） | 通用 Antigravity 全局级 | AGY CLI 专用/暂存级 |
|---|---|---|---|
| MCP | `E:\project\antigravity-use\.agents\mcp_config.json` | `C:\Users\ward\.gemini\config\mcp_config.json` | 无独立 MCP 配置根；CLI 使用项目级和通用全局 MCP |
| Skills | `E:\project\antigravity-use\.agents\skills\` | `C:\Users\ward\.gemini\config\skills\<skill-folder>\` | `C:\Users\ward\.gemini\antigravity-cli\skills\`；CLI 也支持其中的扁平 `<skill-name>.md` 形式 |
| Plugins | `E:\project\antigravity-use\.agents\plugins\` 或 `_agents\plugins\` | `C:\Users\ward\.gemini\config\plugins\` | `C:\Users\ward\.gemini\antigravity-cli\plugins\<plugin_name>\` |
| Hooks | `E:\project\antigravity-use\.agents\hooks.json` | `C:\Users\ward\.gemini\config\hooks.json` | 插件根目录 `hooks.json`；也可由 CLI 主 `C:\Users\ward\.gemini\antigravity-cli\settings.json` 配置 |
| Rules | `E:\project\antigravity-use\.agents\rules\`（兼容 `.agent\rules\`） | `C:\Users\ward\.gemini\GEMINI.md` | 插件根目录 `rules\` |
| CLI 程序/运行数据 | 不适用 | 不适用 | 程序 `C:\Users\ward\AppData\Local\agy\bin\agy.exe`；运行数据 `C:\Users\ward\.gemini\antigravity-cli\`（含 `scratch`、`conversations`、`log` 等） |

共享的是项目级入口和 MCP 读取关系；差异主要在全局 Skill/Plugin/Rule 的目录和包装格式。通用 Skills 优先采用 `<skill-folder>\SKILL.md`，AGY CLI 的全局 slash command 可采用专用目录中的扁平 Markdown。

## 获准执行后的步骤合同（本轮不执行）

1. **基线与能力冻结**：读取仓库指令、`rtk git status --short`、已有 diff、未跟踪文件；重新运行 `agy models`、`agy mcp list`、`agy plugin list`，冻结实际可用扩展。允许读取仓库与上述配置；禁止写入。
2. **只读任务分析**：使用实时最低成本模型，以 `--mode plan --output-format json --print` 调用 AGY。合同必须写明目标、上下文、可用扩展、允许读取、允许写入（`无`）、保持不变、验收和返回字段。验收由主模型检查 JSON 的 `status/error/denied_actions/response`，不得把 AGY 总结当唯一证据。
3. **实现步骤（仅在用户另行明确要求实际执行时）**：按分析结果拆成独立步骤；每步使用 `--mode accept-edits`，允许写入范围精确到文件；默认串行。每步保存 `conversation_id`，检查完整 diff、绝对路径、测试和用户原有改动。
4. **返工与收口**：非空 `denied_actions`、空 `response`、非零退出码、越界产物或验收失败均视为未完成；最多用原会话修正两轮，仍失败由主模型接管最小修改。所有步骤通过后再运行最全面相关验证。

## 只检查与需明确要求的操作

只检查：读取指令/配置、`git status` 与 diff、`agy models`、`agy mcp list`、`agy plugin list`、列出 Skills/Rules/Hooks、解析脱敏后的状态、规划 AGY 命令和验收条件。

需要用户明确要求后才执行：`agy mcp add/remove/enable/disable`；`agy plugin install/disable/enable/uninstall`；修改任意 `mcp_config.json`、`hooks.json`、`settings.json`、`GEMINI.md` 或 Skill/Plugin/Rule；扩大权限、改变 sandbox/权限模式、连接新的外部服务；以及任何真实项目文件实现、提交、推送、发布或不可逆外部操作。本轮用户已明确限定“只输出执行前方案”，因此这些操作全部推迟。

## 当前结论

执行前方案已形成；本轮没有调用 AGY 进行项目变更。真正开始执行前必须先完成 Antigravity 登录并重新运行 `agy models`，取得实时 slug 后才能生成有效的 AGY 步骤合同。
