# AGY 执行前方案

## 1. 本轮边界与触发判断

用户明确要求使用 AGY 完成多步骤项目任务，满足 `agy-orchestrator` 的触发门；但本轮又明确要求“只输出执行前方案”。因此本轮不调用 AGY 执行项目任务，不安装、卸载、启停或连接扩展，不修改配置，不修改任何项目文件；唯一允许写入的是本文件。

工作区基线（只读复测）：分支为 `master`；已有修改和未跟踪内容均属于用户工作，尤其是 `docs/superpowers/*`、`skills/agy-orchestrator/*`、`skills/agy-orchestrator-workspace/*`。若之后获准执行，必须先保存这份基线，只比较本次新增 diff。

## 2. 主编排模型如何发现并复用扩展

发现与复用顺序如下：

1. 先读取仓库指令、当前分支、`git status --short`、已有 diff、未跟踪文件、测试入口，以及本 Skill 要求的 `model-routing.md`、`orchestration-protocol.md`、`agy-extensions.md`。
2. 从项目级入口开始检查：
   - `.agents/mcp_config.json`
   - `.agents/skills/` 中的 `<skill-folder>/SKILL.md`，并兼容扁平 `.md`
   - `.agents/plugins/` 与 `_agents/plugins/`
   - `.agents/hooks.json`
   - `.agents/rules/`，并兼容旧 `.agent/rules/`
3. 只有任务确实需要跨项目能力时，才读取通用 Antigravity 的全局目录：MCP 使用 `C:\Users\ward\.gemini\config\mcp_config.json`；Skills 使用 `C:\Users\ward\.gemini\config\skills\`；Plugins 使用 `C:\Users\ward\.gemini\config\plugins\`；Hooks 使用 `C:\Users\ward\.gemini\config\hooks.json`；Rules 使用 `C:\Users\ward\.gemini\GEMINI.md`。
4. 用运行态确认可复用性，而不把“文件存在”当成“已加载”：运行 `agy mcp list`、`agy plugin list`；必要时在 TUI 使用 `/mcp` 和 `/hooks`。仅将已连接、未禁用、当前步骤确实需要的能力写入 AGY 合同；只在配置中发现但运行态未确认的扩展标为“已配置，运行态待验证”。
5. 每个 AGY 步骤前重新运行 `agy models`，只从当次实时返回的 slug 选择足够完成工作的最低成本模型。模型切换必须记录原因；不得猜测快照名称或静默降级。
6. 对插件提供的同名 Skill、Rule、MCP、Hook 去重并保留插件来源。读取配置时只传递步骤所需的最小信息，隐藏 `env`、HTTP headers、OAuth 客户端信息和 token 的值。插件的安装目录与源目录分开记录；已导入或暂存不等于组件已激活。

发现后的复用方式是：先冻结每一步的扩展清单，再把名称、来源、作用和权限影响写进步骤合同；执行后检查实际工具调用、Hook 输出、拒绝动作、产物绝对路径和 diff，不能只采信 AGY 的文字总结。

## 3. 本机只读发现结果

- AGY CLI 可执行文件：`C:\Users\ward\AppData\Local\agy\bin\agy.exe`；`Get-Command agy` 和 `where.exe agy` 均解析到此路径。日志显示 CLI 版本 `1.2.2`。
- 未发现名为 `antigravity` 的独立 PATH 命令；“通用 Antigravity”是产品/配置范围，不是本机另一个已确认的 CLI 可执行文件。
- 项目级 `E:\project\antigravity-use\.agents\` 当前不存在，因此没有可注入合同的项目级 MCP、Skill、Plugin、Hook 或 Rule。
- 通用全局 MCP 配置发现 `codegraph`；`agy mcp list` 显示 `stdio / enabled`。真正委派前仍需按步骤需要性确认。
- 全局插件目录存在多个插件目录；`agy plugin list` 的运行态导入包含 `ponytail`（skills、commands）、`superpowers`（skills、hooks）、`agent-skills`（skills、agents、commands）和 `i-have-adhd`（skills）。目录存在或被导入，不代表每个组件都与当前步骤相关或已激活。
- 全局开放标准 Skills 根目录 `C:\Users\ward\.gemini\config\skills\` 存在；AGY CLI 专用全局目录 `C:\Users\ward\.gemini\antigravity-cli\skills\` 本次未发现。
- 全局 Rules 文件 `C:\Users\ward\.gemini\GEMINI.md` 存在。插件根目录中的 `rules\` 只有在对应插件实际加载且适用于步骤时才纳入合同。
- 全局 Hooks 文件 `C:\Users\ward\.gemini\config\hooks.json` 存在。CLI 日志报告从 1 个 hooks 文件加载 2 个 named hooks；配置包含 Bash 的 `PreToolUse`（`dcg`），以及 `orca-status` 的 `PreInvocation`、`PostInvocation`、`Stop` 和全工具 `PostToolUse`。Hook 可能注入、拒绝或生成文件，均须进入验收。
- `agy models` 本次因未登录失败并提示先登录；因此当前没有可合法选择的实时模型 slug。日志另有 CLI 日志/崩溃目录访问告警；这不是本轮配置修复目标。

## 4. AGY CLI 与通用 Antigravity 的精确路径差异

| 组件 | 项目级入口（两者共享） | 通用 Antigravity 全局级 | AGY CLI 专用或暂存级 |
|---|---|---|---|
| MCP | `E:\project\antigravity-use\.agents\mcp_config.json` | `C:\Users\ward\.gemini\config\mcp_config.json` | 无独立 MCP 配置根；CLI 使用项目级与通用全局 MCP |
| Skills | `E:\project\antigravity-use\.agents\skills\` | `C:\Users\ward\.gemini\config\skills\<skill-folder>\SKILL.md` | `C:\Users\ward\.gemini\antigravity-cli\skills\`；CLI 文档也支持其中的扁平 `<skill-name>.md` |
| Plugins | `E:\project\antigravity-use\.agents\plugins\` 或 `E:\project\antigravity-use\_agents\plugins\` | `C:\Users\ward\.gemini\config\plugins\` | `C:\Users\ward\.gemini\antigravity-cli\plugins\<plugin_name>\` |
| Hooks | `E:\project\antigravity-use\.agents\hooks.json` | `C:\Users\ward\.gemini\config\hooks.json` | 插件根目录 `hooks.json`；CLI 也可由 `C:\Users\ward\.gemini\antigravity-cli\settings.json` 配置 |
| Rules | `E:\project\antigravity-use\.agents\rules\`（兼容 `.agent\rules\`） | `C:\Users\ward\.gemini\GEMINI.md` | 插件根目录 `rules\` |
| CLI 程序与运行数据 | 不适用 | 不适用 | 程序：`C:\Users\ward\AppData\Local\agy\bin\agy.exe`；运行数据：`C:\Users\ward\.gemini\antigravity-cli\`（含 `settings.json`、日志、会话和暂存数据） |

关键差异：项目级入口相同；MCP 仍从项目级和通用全局配置读取；全局 Skills、Plugins、Rules 的目录不同；AGY CLI 另有自己的运行数据、插件暂存目录和 CLI 专用 Skills 入口。通用开放标准 Skill 优先采用 `<skill-folder>\SKILL.md`，AGY CLI 的全局 slash command 可采用专用目录中的扁平 Markdown 文件。

## 5. Rules 的四种官方激活方式

1. **Manual**：用户通过 `@mention` 手动激活 Rule。
2. **Always On**：Rule 始终应用。
3. **Model Decision**：模型根据 Rule 的自然语言 `description` 判断是否应用。
4. **Glob**：Rule 按文件模式应用，例如 `*.js`、`src/**/*.ts`。

Rule 文件每个最多 12,000 字符；可以用 `@filename` 引用其他文件。相对路径相对 Rule 文件解析；绝对路径先按真实绝对路径解析，目标不存在时再尝试仓库相对路径。编排合同只写当前步骤实际激活的 Rule，不把发现到的全部 Rule 注入上下文。

## 6. 获准执行后的多步骤合同（本轮不执行）

1. **基线与扩展冻结**：重新读取仓库指令、状态、diff、未跟踪文件和测试入口；重新运行 `agy models`、`agy mcp list`、`agy plugin list`，冻结实际运行态。只读，无写入。
2. **只读任务分析**：使用实时最低成本模型，以 `--mode plan --output-format json --print` 调用 AGY。合同必须包含目标、上下文、可用扩展、允许读取、允许写入（无）、保持不变、验收和返回字段。主模型验收 JSON 的 `status`、`error`、`denied_actions`、`response`，不把报告当唯一证据。
3. **顺序实现**：分析通过后按独立结果拆分步骤；每步以 `--mode accept-edits --output-format json --print` 调用，允许写入范围精确到文件，默认串行。保存 `conversation_id`，主模型检查完整 diff、绝对路径、测试和用户原有改动。
4. **返工与收口**：非零退出码、`status: ERROR`、非空 `denied_actions`、空 `response`、越界产物或验收失败均视为未完成；使用原 `conversation_id` 返工，同一缺陷最多两轮，第三次由主模型接管最小必要修改。全部步骤通过后再做全面相关验证。

每一步合同固定写明：目标、上下文、可用扩展、允许读取、允许写入、保持不变、验收、返回。只有当步骤无数据/接口依赖且写入集合完全不相交时才考虑并行；否则串行。

## 7. 只检查与需用户明确要求的操作

**本轮可做且已做的只读检查**：读取指令与 Skill 参考；读取配置的名称、字段和脱敏状态；`git status`、分支、diff 盘点；`agy models`、`agy mcp list`、`agy plugin list`；列出 Skills、Rules、Hooks；规划 AGY 命令和验收条件。

**必须等用户明确要求后才做**：`agy mcp add/remove/enable/disable`；`agy plugin install/disable/enable/uninstall`；修改任何 `mcp_config.json`、`hooks.json`、`settings.json`、`GEMINI.md`、Skill、Plugin 或 Rule；扩大权限、改变 sandbox/权限模式、连接新的外部服务；以及真实项目文件实现、提交、推送、发布或其他不可逆外部操作。本轮“只输出执行前方案”已将这些操作全部推迟。

## 当前结论

执行前方案已完成并写入本文件；本轮没有启动 AGY 项目执行，也没有修改配置。若用户下一步授权实际执行，先完成 Antigravity 登录并重新运行 `agy models`，取得实时 slug 后再生成有效步骤合同；在此之前不能猜测模型或声称任务可执行。
