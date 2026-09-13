# AGY 编排方案：共享配置 → 后端 → 前端

## 编排结论

这三个写入子任务不并行执行，顺序固定为：

1. 修改并验收共享配置。
2. 基于已冻结的配置契约修改并验收后端模块。
3. 基于已冻结的后端响应契约修改并验收前端组件。

原因是后端读取共享配置，前端又依赖后端返回，存在明确的数据与接口依赖；并且“修改同一个配置文件”还造成写入集合冲突。时间压力不改变并行门。并发写入会让后端或前端建立在尚未验收的契约上，也无法可靠归属同一文件中的改动。

仅允许把三个方向的**只读调查**并行化；调查结果必须先由主模型汇总，确定精确文件、测试入口并冻结配置/API 契约，之后才进入上述串行写入阶段。本方案本身不调用 AGY 写文件，也不修改任何项目文件。

## 执行前硬门

1. 记录仓库指令、当前分支、`git status --short`、已有完整 diff、未跟踪文件及相关测试状态，形成 `BASELINE`。当前基线已有用户改动，执行时全部视为受保护内容，不覆盖、不重排、不顺手整理。
2. 每次准备调用 AGY 前运行 `agy models`，只从该次实时输出中选足以完成步骤的最低成本模型 slug。当前预检未登录，实时列表为空，所以正式执行必须先恢复 AGY 登录并重新运行该命令；不使用静态快照猜测 slug。
3. 通过只读搜索解析并填入以下精确值：
   - `CONFIG_FILE`：唯一共享配置文件；
   - `BACKEND_FILE_SET`：读取该配置的后端实现及对应测试；
   - `FRONTEND_FILE_SET`：消费后端响应的前端组件及对应测试；
   - `CONFIG_TEST`、`BACKEND_TEST`、`FRONTEND_TEST`、`E2E_CHECK`：仓库已有的最小相关验证命令。
4. 确认三个写入集合精确且互不混用：步骤 1 只写 `CONFIG_FILE`，步骤 2 只写 `BACKEND_FILE_SET`，步骤 3 只写 `FRONTEND_FILE_SET`。任何集合仍不明确时，继续只读调查，不启动写入。

可选的并行只读调查均使用 `--mode plan --output-format json`，分别查清配置结构、后端读取链和前端消费链。主模型审阅三个 JSON 结果及代码实况后，产出并冻结 `CONFIG_CONTRACT_V1`、`BACKEND_RESPONSE_CONTRACT_V1` 的预期草案；调查代理无写入权限。

## 步骤 1：配置

调用形式：

```powershell
agy --model <LIVE_MODEL_SLUG> --mode accept-edits --output-format json --print "<CONFIG_STEP_PROMPT>"
```

步骤合同：

```text
目标：在 CONFIG_FILE 中完成需求对应的最小配置变更，并给出明确的字段名、类型、默认值、必填性和兼容行为。
上下文：BASELINE；主模型汇总后的只读调查；当前配置格式和加载约定。
允许读取：CONFIG_FILE、配置加载代码、配置 schema/类型定义、相关配置测试与示例。
允许写入：仅 CONFIG_FILE（若 schema 与配置必须原子更新，应在开始前把精确 schema 文件加入本步骤，且从后续步骤写入集合中移除）。
保持不变：BASELINE 中的用户已有改动；BACKEND_FILE_SET、FRONTEND_FILE_SET；所有范围外文件；无关配置键与公开接口。
验收：运行配置解析/校验命令 CONFIG_TEST；检查缺省值、无效值和兼容输入；确认 diff 仅落在允许写入范围。
返回：完成摘要、检查文件、修改文件、命令与结果、最终 CONFIG_CONTRACT_V1、遗留问题、越界情况。
```

主模型审查门：解析 JSON 的 `status`、`error`、`response` 并保存 `conversation_id`；相对 `BASELINE` 列出新增变更；阅读完整 diff；核对范围、边界条件和最小性；亲自运行 `CONFIG_TEST`；核对报告与真实工作区；确认用户原改动完整。六项均通过后才冻结 `CONFIG_CONTRACT_V1` 并进入步骤 2。

## 步骤 2：后端

调用形式同上，但使用新的步骤会话和后端合同。

```text
目标：让后端按已验收的 CONFIG_CONTRACT_V1 读取配置，并以明确、稳定的响应契约返回前端所需数据。
上下文：BASELINE；已验收且不可自行改写的 CONFIG_CONTRACT_V1；当前后端技术栈与错误处理约定。
允许读取：CONFIG_FILE、BACKEND_FILE_SET、后端路由/服务/类型定义及相关测试；只读查看前端消费点以确认契约。
允许写入：仅预先列明的 BACKEND_FILE_SET。
保持不变：CONFIG_FILE、CONFIG_CONTRACT_V1、FRONTEND_FILE_SET、用户已有改动、无关后端接口和范围外文件。
验收：运行 BACKEND_TEST；覆盖正常配置、缺省配置、非法配置及错误映射；断言响应字段、类型、可空性和状态码符合 BACKEND_RESPONSE_CONTRACT_V1；确认 diff 未越界。
返回：完成摘要、检查文件、修改文件、命令与结果、最终 BACKEND_RESPONSE_CONTRACT_V1、遗留问题、越界情况。
```

主模型重复完整六项审查门并亲自运行 `BACKEND_TEST`。通过后冻结 `BACKEND_RESPONSE_CONTRACT_V1`，再进入步骤 3；后端步骤不得回写配置来迁就实现。

## 步骤 3：前端

调用形式同上，但使用新的步骤会话和前端合同。

```text
目标：让 FRONTEND_FILE_SET 中的组件正确消费已验收的 BACKEND_RESPONSE_CONTRACT_V1，并处理加载、成功、空值和错误状态。
上下文：BASELINE；不可自行改写的 CONFIG_CONTRACT_V1 与 BACKEND_RESPONSE_CONTRACT_V1；当前前端框架、类型与组件测试约定。
允许读取：FRONTEND_FILE_SET、前端 API 客户端/共享类型、组件测试；只读查看已验收后端响应定义。
允许写入：仅预先列明的 FRONTEND_FILE_SET。
保持不变：CONFIG_FILE、BACKEND_FILE_SET、两个已冻结契约、用户已有改动、无关 UI 和范围外文件。
验收：运行 FRONTEND_TEST 及类型检查；覆盖加载、成功、字段缺失/空值和后端错误状态；确认 diff 未越界且没有复制或重新定义冲突的 API 契约。
返回：完成摘要、检查文件、修改文件、命令与结果、遗留问题、越界情况。
```

主模型再次执行完整六项审查门并亲自运行 `FRONTEND_TEST` 和类型检查。通过后才允许进入全链路验收。

## 失败与返工协议

每个步骤首次调用后保存该步骤自己的 `conversation_id` 与 `LIVE_MODEL_SLUG`。审查失败时不新开会话，而是把具体 diff、测试失败输出、越界文件和未改变的验收条件发回同一个会话：

```powershell
agy --conversation <SAME_CONVERSATION_ID> --model <SAME_LIVE_MODEL_SLUG> --mode accept-edits --output-format json --print "<具体失败证据 + 原允许写入范围 + 原验收条件>"
```

返工后从 JSON 检查开始，重新执行全部六项主模型审查门。同一缺陷最多返工两轮；第二轮仍未通过时，停止该 AGY 会话，由主模型接管最小必要修改并重新验证。当前步骤未通过期间，下游步骤保持未启动状态。

若 AGY 非零退出或返回 `status: ERROR`，先检查 stderr、`error` 和是否发生部分写入；所有部分写入也必须按基线审查并归入该步骤，不能把 `SUCCESS`、退出码或 AGY 的文字总结当作完成证据。

## 最终验收与交付

三个步骤分别通过后，由主模型：

1. 相对 `BASELINE` 阅读完整最终 diff，确认只包含三个合同授权的文件，用户已有改动保持完整。
2. 依次重跑 `CONFIG_TEST`、`BACKEND_TEST`、`FRONTEND_TEST`、类型检查，再运行 `E2E_CHECK` 验证配置到后端响应再到前端渲染的真实链路。
3. 核对最终实现仍符合冻结的 `CONFIG_CONTRACT_V1` 与 `BACKEND_RESPONSE_CONTRACT_V1`，并检查兼容性、错误路径和最小性。
4. 交付时分别列出完成内容、实际运行的验证及结果、仍存在的限制；AGY 报告仅作辅助记录，不作唯一证据。

因此，赶时间的优化点放在并行只读调查、精确缩小写入范围和选择最低成本实时模型上，而不是并行修改存在共享文件及上下游依赖的三层代码。
