# AGY 编排 Skill 设计

## 目的

创建一个 Codex Skill：让具备较强能力的主编排模型通过 `agy` 命令行调用
Google Antigravity，把较长、可拆分的任务交给 AGY 执行，同时由主模型负责
范围控制、逐步审查、返工和最终验证。

该 Skill 优先面向使用 `gpt-5.6-sol` 或 `gpt-6-astra` 的 Codex 会话。用户必须在
提示词中明确要求使用 Antigravity/AGY，或直接调用 `$agy-orchestrator`，然后再判断
任务是否包含多个有意义且可单独验收的步骤。小型修改和简短问答仍由主模型直接完成。

## 交付内容

Skill 放在 `skills/agy-orchestrator/`，包含：

- `SKILL.md`：触发条件和编排流程。
- `agents/openai.yaml`：可被发现的 UI 元数据，并开启正常发现；description 中的显式
  意图与复杂度门槛负责阻止误触发。
- `references/model-routing.md`：当前模型清单、模型任务分工和运行时刷新规则。
- `references/orchestration-protocol.md`：AGY 命令模式、返回协议、审查门、返工和
  失败恢复规则。
- `references/agy-extensions.md`：MCP、Skills、Plugins、Hooks、Rules 的项目级与全局级
  发现、路径差异和复用边界。

初始版本不需要额外的包装程序。原生 CLI 已提供模型发现、JSON 信封、流式事件和
会话恢复能力。把判断保留在主模型中，也能避免脚本把“命令执行成功”误判成“任务
已经完成”。

## 触发条件

以下条件同时满足时使用本 Skill：

1. 用户在提示词中明确要求使用 Antigravity/AGY，或直接调用 `$agy-orchestrator`。
2. 任务至少可以拆成两个有明确验收条件的步骤。
3. 委派可以降低执行成本或延迟，同时不削弱审查。
4. 委派工作仍属于当前任务的范围和权限。

`gpt-5.6-sol` 和 `gpt-6-astra` 是优先的主编排模型，但不作为硬触发条件。仅在用户
表达“使用 Antigravity/AGY”的执行意图或直接调用本 Skill 时进入触发判断；讨论产品、
询问文档或提到模型名称不算执行意图。简单修改、需要先向用户澄清的工作，或主模型
无法检查结果的工作，留在主模型处理。

## 运行时发现与模型路由

每次调用 Skill 开始时运行 `agy models`。命令返回的实时列表是当前可用模型 slug
的唯一依据，Skill 中的带日期快照只用于对照，不作为实时选择依据。

2026-09-13 本机 CLI 返回的初始模型清单如下，每一项都会在路由参考中单独记录：

1. `gemini-3.8-flash-high`
2. `gemini-3.8-flash-medium`
3. `gemini-3.8-flash-low`
4. `gemini-3.7-flash-high`
5. `gemini-3.7-flash-medium`
6. `gemini-3.7-flash-low`
7. `gemini-3.6-flash-high`
8. `gemini-3.6-flash-medium`
9. `gemini-3.6-flash-low`
10. `gemini-3.1-pro-high`
11. `gemini-3.1-pro-low`
12. `claude-sonnet-4-6`
13. `claude-opus-4-6-thinking`
14. `gpt-oss-120b-medium`

新发现的模型要根据其显示的模型家族、推理档位、可查到的官方特点和任务实际需求
重新判断。已从列表移除的模型不得继续选择。最新的合适 Flash 模型是默认实现工
作者；旧版 Flash 只作为回退，默认顺序排在最新版本之后。

## 编排流程

### 1. 建立基线

委派前，主模型检查仓库规则、当前状态、已有改动和相关测试。用户已有的未提交改动
保持原样。

### 2. 拆分任务

主模型创建有顺序的步骤。每一步都要定义：

- 一个具体结果；
- 允许修改的文件范围；
- 相关上下文和约束；
- 可观察的验收条件；
- 对前置步骤的依赖。

默认串行执行。只有步骤之间没有依赖，且允许写入的文件集合完全不相交时才允许并
行。文件归属不确定时，按串行执行。

### 3. 选择 AGY 模型

选择足以胜任该步骤的最低成本实时模型：

- 最新 Flash Low：搜索、机械修改、格式化和小型测试。
- 最新 Flash Medium：普通实现和大多数聚焦型多文件工作。
- 最新 Flash High：调试或需要更深推理的实现。
- Gemini Pro：架构、迁移、长上下文或困难规划。
- Claude Sonnet：平衡速度和质量的实现、审查与文档。
- Claude Opus：困难推理、长流程和复杂重构。
- GPT-OSS 120B：文本型推理、结构化分析和独立第二意见。

详细路由参考会逐项说明当前可用配置。

### 4. 执行单个步骤

使用 AGY headless 模式，显式传入模型、工作目录和提示词，并使用
`--output-format json`。已批准的实现步骤使用 `--mode accept-edits`，只读分析使用
`--mode plan`。默认不启用 `--dangerously-skip-permissions`。

提示词要求 AGY 返回：

- 执行结果；
- 检查过和修改过的文件；
- 验证命令及结果；
- 尚未解决的问题；
- 是否超出分配范围。

主模型保存返回的 `conversation_id` 以便返工。退出码为 0 或 `status: SUCCESS` 只
代表传输成功；任务是否完成仍以主模型的验收为准。

### 5. 每一步之后审查

进入下一步前，主模型独立验证：

- 结果是否符合步骤目标；
- 是否只修改了允许的文件；
- diff 是否技术正确且足够精简；
- 相关测试或检查是否真实通过；
- AGY 摘要是否与工作区实际状态一致；
- 是否损坏用户原有改动。

主模型必须亲自检查产物和 diff。AGY 自己的总结属于辅助证据，最终验收结论由主模型
给出。

### 6. 返工或接管

如果审查失败，使用 `--conversation <id>` 恢复原会话，发送具体差异和不变的验收
条件，然后重新审查结果。

同一个缺陷最多允许 AGY 修正两轮。如果仍不合格，或者返工扩大了范围，主模型接管，
进行最小必要修改并运行验收检查。

### 7. 最终审查

所有步骤分别通过后，主模型对完整 diff 做整体验收，并运行最全面的相关验证。只
有完成这一步后，才能向用户报告完成。

## AGY 返回信息处理

默认使用 `--output-format json` 返回的单个 JSON 信封：

- `conversation_id`：返工句柄。
- `status`：CLI 终态。
- `response`：工作模型报告。
- `error`：失败时的详细信息。
- `duration_seconds`、`num_turns`：运行元数据。
- `usage`：token 统计。

当需要实时观察工具调用和步骤进度时，可以使用 `--output-format stream-json`。最终
仍以其中的 `result` 事件作为完整结果信封。

初始版本不依赖 `--json-schema`：实测发现它与本机某个工具定义组合存在兼容性错
误。同时不假设 `agy models --output-format json` 一定存在：本机 CLI `1.2.2` 对该
参数返回了 flag 错误，尽管较新的在线文档已经描述了机器可读列表输出。

## 错误处理

- 模型未知：刷新 `agy models`，选择另一个合适的实时 slug；模型家族切换需明确记录。
- 身份验证失败：报告准确原因并保留所有已有工作。
- 非零退出码或 `status: ERROR`：先检查 stderr 和 JSON 的 `error` 字段，再判断 response
  是否具备可用价值。
- 超时但有部分输出：检查工作区状态和 stderr 警告，再决定恢复还是重试。
- 权限被拒绝：保留被拒绝的动作记录，并由主 Codex 会话处理所需的用户确认。
- 超出范围：拒绝该步骤，只通过定向修改处理 AGY 自己产生的越界改动，同时保留之
  前已有工作。

## 验证策略

完成后的 Skill 通过四项检查：

1. 对 Skill 目录运行 Codex Skill 校验器。
2. 确认路由参考包含 `agy models` 当前返回的每个模型。
3. 在隔离目录运行一个小型 AGY 实现任务，检查 JSON 信封、文件 diff 和测试结果。
4. 通过 `--conversation` 发送一条刻意设计的审查意见，验证返工后的产物，再执行
   最终验收。

## 来源

- Google Antigravity CLI headless 模式：
  https://antigravity.google/docs/cli/headless/
- Google Antigravity 模型：
  https://antigravity.google/docs/models/
- Google Antigravity CLI 功能：
  https://antigravity.google/docs/cli/features/
- Gemini 3.8 Flash 发布说明：
  https://antigravity.google/blog/gemini-3-8-flash-in-google-antigravity
- Gemini 3.1 Pro 发布说明：
  https://antigravity.google/blog/gemini-3-1-pro-in-google-antigravity
- Anthropic 模型总览：
  https://docs.anthropic.com/en/docs/about-claude/models
- OpenAI GPT-OSS 120B 模型页：
  https://developers.openai.com/api/docs/models/gpt-oss-120b
