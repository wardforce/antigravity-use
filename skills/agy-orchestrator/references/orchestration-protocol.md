# AGY 编排协议

## 运行前基线

记录仓库指令、当前分支、`git status --short`、已有 diff、未跟踪文件和相关测试状态。
AGY 返回后始终与该基线比较，以便保留用户原有工作并识别本步骤新增变更。

## 步骤合同

给 AGY 的每一步提示词都应包含以下字段：

```text
目标：一个可验收结果。
上下文：上一步已验收的契约和当前技术栈。
允许读取：完成任务所需的目录或文件。
允许写入：精确文件或边界清楚的路径集合。
保持不变：用户已有改动、公开接口和范围外文件。
验收：AGY 应运行的具体测试或检查。
返回：完成摘要、检查文件、修改文件、命令与结果、遗留问题、越界情况。
```

## 新步骤调用

只读分析：

```powershell
agy --model <LIVE_MODEL_SLUG> --mode plan --output-format json --print "<STEP_PROMPT>"
```

文件实现：

```powershell
agy --model <LIVE_MODEL_SLUG> --mode accept-edits --output-format json --print "<STEP_PROMPT>"
```

默认保留 AGY 的权限审查，不添加 `--dangerously-skip-permissions`。命令从目标仓库根目录
运行；需要额外工作区时显式使用 `--add-dir`。

## JSON 结果

默认解析一个 JSON 信封：

| 字段 | 用途 |
|---|---|
| `conversation_id` | 保存为该步骤的返工句柄 |
| `status` | 判断 CLI 终态，不代替内容验收 |
| `response` | AGY 的执行报告，与工作区实况交叉检查 |
| `error` | 失败诊断 |
| `denied_actions` | headless 自动拒绝的动作；非空即表示步骤未完成 |
| `duration_seconds` | 耗时记录 |
| `num_turns` | 确认是否在原会话返工 |
| `usage` | token 使用记录 |

非零退出码、`status: ERROR`、非空 `denied_actions` 或空 `response` 都进入失败处理。
先检查 stderr、错误字段和是否存在部分写入，再决定恢复会话或由主模型接管。

## 主模型审查门

每个步骤返回后按固定顺序检查：

1. 比较当前状态与执行前基线，列出本步骤新增变更。
2. 阅读完整 diff，确认所有写入均在步骤合同范围内。
3. 检查行为、接口、边界条件和最小性。
4. 主模型亲自运行合同中的测试或检查，不只引用 AGY 的报告。
5. 核对 AGY 报告中的文件和命令与真实工作区一致。
6. 确认用户原有改动保持完整。

AGY 报告中的绝对路径也要检查。目标在项目目录而实际产物位于 AGY 自身 `scratch`
时，按越界处理。

只有六项均通过，才将产物和接口契约交给下一步。

## 同会话返工

使用第一次返回的 `conversation_id`：

```powershell
agy --conversation <CONVERSATION_ID> --model <SAME_LIVE_MODEL_SLUG> --mode accept-edits --output-format json --print "<REVIEW_FEEDBACK>"
```

返工提示包含具体失败证据、允许写入范围和原验收条件。返工后从头执行主模型审查门。
同一缺陷最多两轮；第三次处理由主模型完成最小修改。

## 并行判断

| 检查 | 结果 |
|---|---|
| 存在数据、接口或生成物依赖 | 串行 |
| 两步可能修改同一文件 | 串行 |
| 写入范围尚未确认 | 串行 |
| 无依赖且写入集合完全不相交 | 可以并行 |

只读调查可以并行，但其结果要先由主模型汇总并冻结合同，再开始写入步骤。

## 流式观测

需要查看实时工具调用时可用：

```powershell
agy --model <LIVE_MODEL_SLUG> --mode accept-edits --output-format stream-json --print "<STEP_PROMPT>"
```

依次处理 `init`、`step_update` 和最终 `result`。最终判断仍读取 `result.status`，再进入
主模型审查门。

Windows 上通过 PowerShell 管道使用 `--input-format stream-json` 时可能出现 UTF-8 BOM，
因此默认用 `--print` 启动单轮，再用 `--conversation` 返工。

本机实测 `--sandbox` 可能让 AGY 只看到自己的 `scratch`，并把相对路径产物写到那里。
需要修改项目文件时不把 `--sandbox` 当作路径正确性的保证；始终核对实际绝对路径。
headless 权限拒绝时，优先缩小动作、复用原会话或由主模型接管。只有用户在了解范围后
明确批准，才考虑改变 AGY 权限模式。

## 当前 CLI 兼容记录

2026-09-13 对 AGY CLI 1.2.2 的本机实测：

- `text`、`json`、`stream-json` 三种输出均可用。
- `agy models` 和 `agy agents` 返回文本列表；本机构建不接受它们的
  `--output-format json` 参数。
- `--json-schema` 与当前工具定义组合曾返回 provider 400，初始协议不依赖该参数。
- 未识别的模型 slug 会非零退出并列出可用模型。
- `--conversation <id>` 可以恢复原会话，返工后的 `num_turns` 会增长。
- headless 权限拒绝可能仍返回 `status: SUCCESS`，同时给出空 `response` 和
  `denied_actions`；必须按失败处理。
- `--sandbox` 探针曾把目标文件写到 AGY `scratch`，实际绝对路径必须进入验收。

## 完整示例

任务是依次修改数据库字段、后端接口和前端表单时：

1. 运行 `agy models`，记录用户改动基线。
2. 委派数据库步骤；主模型审查迁移 diff 并运行迁移测试。
3. 通过后冻结字段契约，委派后端步骤；主模型审查 API diff 和接口测试。
4. 通过后冻结响应契约，委派前端步骤；主模型审查 UI diff、类型检查和组件测试。
5. 最后由主模型运行端到端验证并检查完整 diff。

任一步未通过都留在当前步骤返工，不让下游基于未验收契约继续工作。

## 官方依据

- Headless 与返回格式：https://antigravity.google/docs/cli/headless/
- CLI 功能与子代理：https://antigravity.google/docs/cli/features/
- CLI 参考：https://antigravity.google/docs/cli/reference/
