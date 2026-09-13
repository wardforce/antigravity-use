# Antigravity 两分钟六文件重构编排方案

## 编排结论

本方案只描述正式执行流程，不调用 AGY，也不修改六个目标文件。两分钟是优化目标，不是降低验收标准的理由。

正式执行时必须先运行 `agy models`，只从该次实时返回的 slug 中选择足以完成步骤的最低成本模型，并记录为 `LIVE_MODEL_SLUG`。记忆中的“Gemini 3.8 Flash”及技能文档里的历史快照都不作为当前可用依据；若实时列表包含合适的 3.8 Flash 档位，再根据任务复杂度选择。若没有，则从实时列表重新选型，不猜测别名。

三个写入任务只有在主模型已经证明它们没有数据、接口或生成物依赖，且写入集合完全不相交时才同时启动。六个文件先记为：

- `TASK_A_FILES = {FILE_1, FILE_2}`
- `TASK_B_FILES = {FILE_3, FILE_4}`
- `TASK_C_FILES = {FILE_5, FILE_6}`

上述分组只是占位，不是预先成立的并行授权。若只读调查发现任意依赖、同一文件可能被多个任务修改，或文件归属仍不明确，则把有关任务按依赖顺序串行执行。可以并行的只读调查必须先结束，由主模型汇总结果、冻结接口契约和精确写入集合，之后才能开始写入。

## 0. 执行前基线与模型发现

在计时开始前或正式执行的第一个时间片内，由主模型完成：

1. 读取仓库指令，记录当前分支、`git status --short`、完整已有 diff、未跟踪文件和相关测试初始状态，形成 `BASELINE`。当前仓库已有用户改动，全部标记为受保护内容。
2. 运行 `agy models`，保存实时输出与时间戳；选定 `LIVE_MODEL_SLUG`。每次新一轮 AGY 调用前都刷新列表。
3. 并行发起最多三个 `--mode plan --output-format json` 的只读调查，分别定位六个目标文件的职责、调用关系、共享接口、测试入口和可能的生成物依赖。只读任务没有任何写入权限。
4. 主模型亲自核对调查结论与代码实况，产出 `REFACTOR_CONTRACT`：外部行为、公开接口、输入输出、错误语义、兼容要求，以及每个文件的唯一所有者。
5. 只有依赖图显示 A、B、C 互不依赖，且三个文件集合两两不相交，才设置 `PARALLEL_WRITES_ALLOWED=true`；否则生成拓扑顺序并串行执行。

两分钟内可压缩的是调查范围、提示词和测试选择，不省略 `agy models`、基线、依赖判定或验收门。

## 1. 三个写入步骤合同

若并行门通过，可同时启动以下三条命令；若未通过，则仍使用相同合同按拓扑顺序逐条启动：

```powershell
agy --model <LIVE_MODEL_SLUG> --mode accept-edits --output-format json --print "<TASK_A_PROMPT>"
agy --model <LIVE_MODEL_SLUG> --mode accept-edits --output-format json --print "<TASK_B_PROMPT>"
agy --model <LIVE_MODEL_SLUG> --mode accept-edits --output-format json --print "<TASK_C_PROMPT>"
```

每个提示词必须完整包含以下字段，并把 A/B/C 与对应文件集合代入：

```text
目标：在 TASK_X_FILES 内完成 REFACTOR_CONTRACT 中属于任务 X 的最小重构，保持对外行为不变。
上下文：BASELINE、已冻结的 REFACTOR_CONTRACT、当前技术栈和仓库约定。
允许读取：六个目标文件、直接调用方/被调用方、相关类型定义和测试文件。
允许写入：仅 TASK_X_FILES 中精确列出的两个文件。
保持不变：BASELINE 中用户已有改动；其他两个任务的文件；公开接口；范围外文件；锁文件与生成物（除非合同预先明确授权）。
验收：运行 TASK_X_TEST；覆盖正常路径、边界条件和错误路径；列出实际修改文件并检查越界写入。
返回：完成摘要、检查文件、修改文件、命令与结果、遗留问题、越界情况。
```

主模型分别保存三个 JSON 信封中的 `conversation_id`、`status`、`error`、`duration_seconds`、`num_turns` 和 `usage`。`status: SUCCESS` 或退出码 0 只表示 AGY 调用结束，不表示重构已完成。

## 2. 每一步的主模型验收门

无论三个任务并行还是串行，每个任务返回后都必须单独经过以下六项检查：

1. 与 `BASELINE` 比较，准确列出该步骤新增的变更。
2. 阅读该步骤完整 diff，确认写入只落在其两个授权文件中。
3. 检查实现行为、公开接口、边界条件、错误路径和修改最小性。
4. 主模型亲自运行 `TASK_X_TEST`，不引用 AGY 的测试摘要代替实际结果。
5. 核对 AGY 报告中的检查文件、修改文件、命令和结果与真实工作区一致。
6. 确认用户原有改动完整保留，且没有跨任务覆盖或生成额外文件。

六项全部通过后，该步骤才标记为 `ACCEPTED`。并行调用中，即使 A、B、C 都返回 `SUCCESS`，任何一个未通过上述审查，整体仍未完成。

## 3. 失败与返工

若某一步越界、测试失败、报告与实况不符，或实现不满足合同，使用该步骤原有的 `conversation_id` 和同一个实时模型 slug 返工：

```powershell
agy --conversation <CONVERSATION_ID> --model <SAME_LIVE_MODEL_SLUG> --mode accept-edits --output-format json --print "<具体失败证据 + 原写入范围 + 原验收条件>"
```

返工提示必须包含真实 diff 或测试输出；返工后从 JSON 解析开始，重新执行全部六项检查。同一缺陷最多交给 AGY 修正两轮；仍未通过时由主模型接管最小必要修改并重新验证。若 AGY 非零退出或返回 `status: ERROR`，先检查 stderr、`error` 和部分写入，并把部分写入也纳入基线审查。

## 4. 最终验收与完成标准

只有三个步骤都为 `ACCEPTED` 后，主模型才执行全局验收：

1. 相对 `BASELINE` 阅读六文件的完整最终 diff，确认没有范围外变更，也没有破坏用户原有改动。
2. 重跑 A、B、C 的相关测试，再运行跨文件集成测试或构建命令 `INTEGRATION_CHECK`。
3. 核对六个文件共同满足 `REFACTOR_CONTRACT`，特别检查跨文件接口、初始化顺序、错误传播和兼容性。
4. 仅当完整 diff、所有相关测试和集成验证均通过，才报告重构完成；否则报告具体未通过项和剩余限制。

因此，最终完成判据是“六文件 diff 经主模型审查且相关测试真实通过”，不是 AGY 的 `SUCCESS`。若两分钟截止时尚未取得这些证据，应如实报告当前已验收步骤与未验收步骤，不把超时转化为跳过验证。
