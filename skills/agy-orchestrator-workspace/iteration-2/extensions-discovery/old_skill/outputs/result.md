# AGY 多步骤项目任务：执行前方案（旧版 Skill 基线）

本轮只输出方案，不调用 AGY、不安装或启停扩展、不修改任何配置。任务满足 AGY 编排门槛的前提是：用户明确要求使用 AGY，且存在至少两个彼此可验收的步骤；主模型能检查产物并且委派不超出用户授权。

## 1. 主模型预检与扩展发现

1. 读取仓库指令和本 Skill 的引用协议；记录当前分支、`git status --short`、已有 diff、未跟踪文件和相关测试入口。所有这些是只读检查，用户原有改动作为不可覆盖的基线。
2. 从项目根目录向上发现项目级入口，优先检查以下路径是否存在、可读以及是否声明了加载/激活规则：
   - MCP：`.agents/mcp_config.json`
   - Skills：`.agents/skills/`
   - Plugins：`.agents/plugins/`
   - Hooks：`.agents/hooks.json`
   - Rules：`.agents/rules/`
3. 检查全局入口，但只读取和汇总，不写回：
   - MCP、通用 Skills、通用 Plugins、通用 Hooks：`~/.gemini/config/`
   - 全局 Rules：`~/.gemini/GEMINI.md`
   - AGY CLI 专属 Skills/Plugins 暂存与内置资源：`~/.gemini/antigravity-cli/`
4. 建立“项目优先、全局补充、CLI 专属按 CLI 语义”的候选清单：去重同名 Skill/Plugin，记录来源、版本/描述、所需工具、权限影响和与当前步骤的相关性。只复用已经发现且与任务匹配的扩展；不能因为目录存在就自动启用全部扩展。
5. 对 MCP 逐项确认服务器名、命令/入口、工作目录、环境变量引用和读写能力；对 Hooks 读取触发事件与脚本，预判它们是否可能改变工具结果、拦截命令或影响权限。凭配置文本不能把“已安装/已发现”说成“已连接/已成功运行”。

正式调用模型前，每次都运行：

```powershell
agy models
```

只从这次实时输出的 slug 选足以完成当前步骤的最低成本模型。历史快照和记忆中的模型名不能替代实时列表；若 slug 无效，重新运行 `agy models`，不猜别名。

## 2. AGY CLI 与通用 Antigravity 的路径差异

以下是本方案中必须区分的逻辑路径（`~` 在本机展开为 `C:\Users\ward`）：

| 能力 | 项目级入口 | 通用 Antigravity 全局入口 | AGY CLI 全局/暂存入口 |
|---|---|---|---|
| MCP | `.agents/mcp_config.json` | `~/.gemini/config/mcp_config.json` | CLI 不把通用全局路径改写成项目路径；先按 CLI 文档检查其实际读取范围，不能猜测替代文件 |
| Skills | `.agents/skills/` | `~/.gemini/config/skills/` | `~/.gemini/antigravity-cli/skills/` |
| Plugins | `.agents/plugins/` | `~/.gemini/config/plugins/` | `~/.gemini/antigravity-cli/plugins/`（CLI 暂存/专属位置，不等同于通用插件目录） |
| Hooks | `.agents/hooks.json` | `~/.gemini/config/hooks.json` | 先检查 CLI 的实际加载说明；不能把通用 Hook 自动当作 CLI Hook |
| Rules | `.agents/rules/` | `~/.gemini/GEMINI.md` | CLI 仍须按其加载规则核对项目/全局 Rules；不能用 CLI 目录中的 Skill 反推 Rules 已激活 |

因此，“通用 Antigravity 已有扩展”与“AGY CLI 本次会加载的扩展”是两个集合。项目入口用于当前仓库；`~/.gemini/config/*` 是通用全局配置；`~/.gemini/antigravity-cli/*` 是 CLI 的 Skills/Plugins 专属或暂存树。发现同名资源时保留来源和优先级证据，不能静默用一个路径覆盖另一个路径。路径存在也不证明 AGY 已加载，必须以 CLI 的加载/调用结果和实际工作区证据核实。

## 3. Rules 与 Hooks 的激活和风险

- 项目 Rules 从 `.agents/rules/` 发现，并按仓库/当前工作目录的适用范围和文件命名约定激活；全局 `~/.gemini/GEMINI.md` 作为用户级通用规则来源。主模型在执行前只读取、解释并记录适用规则，不改写它们。
- 项目/全局 `hooks.json` 中的 Hook 可能在工具调用前后、命令执行前后或会话事件上运行脚本，因此可能改变工具执行结果、阻止动作、注入参数或影响权限判断。主模型要把 Hook 产生的拒绝、额外文件或命令输出纳入验收，不能绕过或将其视为噪声。
- 只有“已发现、明确适用、权限影响可解释”的现有扩展才能进入步骤合同；扩展未加载或权限被拒时，暂停当前步骤并记录证据，不自动放宽权限。

## 4. 串行步骤合同

先由主模型完成只读依赖调查，冻结每一步精确的读取文件、写入文件、测试命令和跨步骤契约。默认按依赖串行；只有无数据/接口依赖且允许写入集合完全不相交时才考虑并行。每次委派提示必须包含：

```text
目标：一个可验收结果。
上下文：已验收的上一步契约和当前技术栈。
允许读取：所需目录或文件。
允许写入：精确文件或边界清楚的路径集合。
保持不变：用户已有改动、公开接口和范围外文件。
验收：具体测试或检查命令。
返回：摘要、检查文件、修改文件、命令与结果、遗留问题、越界情况。
```

对只读分析使用：

```powershell
agy --model <LIVE_MODEL_SLUG> --mode plan --output-format json --print "<STEP_PROMPT>"
```

对获得用户授权后的文件实现使用：

```powershell
agy --model <LIVE_MODEL_SLUG> --mode accept-edits --output-format json --print "<STEP_PROMPT>"
```

不添加 `--dangerously-skip-permissions`；从目标仓库根目录运行，只有确有需要且经授权才使用额外目录。

每个步骤返回后，主模型依次核对：JSON 的 `status`/`error`/`denied_actions`/`response`，相对基线的新增文件和完整 diff，写入是否越界，真实测试/构建/用户可见结果，AGY 报告与工作区是否一致，以及用户原有改动是否完整保留。`status: SUCCESS`、退出码 0 或文字摘要都不是完成证据；空 `response` 或非空 `denied_actions` 直接判为未完成，不进入下游步骤。

若失败，检查 stderr、部分写入和产物绝对路径；有 `conversation_id` 时在同一会话、同一实时模型下提交具体失败证据和不变的验收条件返工。相同缺陷最多返工两轮；仍失败则由主模型接管最小必要修改并重新执行全部相关验收。若报告的产物位于 AGY 自身 `scratch` 而非合同路径，按越界处理。

## 5. 只检查与需明确要求的操作

本次只允许：读取仓库和全局配置、列出/解析已存在的 MCP/Skills/Plugins/Hooks/Rules、运行只读状态检查、运行 `agy models`（若进入正式执行前选型）、制定步骤合同。不会因为发现缺失或不可用而自行修复配置。

以下操作必须等用户明确要求后再做，并在执行前重新确认具体目标与范围：安装、卸载、升级或下载 Plugin/Skill/MCP；启用、停用或修改任何配置；写入/删除/移动 `mcp_config.json`、`hooks.json`、Rules、全局设置或 CLI 暂存目录；改变权限模式、绕过 Hook/权限审查；连接外部服务、写入外部系统或提交不可逆的真实业务操作。AGY 的文件修改本身也只有在用户要求“完成该多步骤项目任务”且每一步合同已冻结后才允许，且仅限合同白名单。

当前结论：扩展发现和路径核对完成后才能选择实时模型并形成正式委派；本轮不启动 AGY、不修改配置、不产生项目代码变更。
