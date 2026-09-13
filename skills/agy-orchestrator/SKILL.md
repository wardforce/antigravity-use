---
name: agy-orchestrator
description: Use when the user explicitly asks Codex to use Antigravity or AGY, or invokes $agy-orchestrator, for work containing at least two independently verifiable steps.
---

# AGY 编排器

## 核心原则

主模型负责拆分、范围、验收和最终结论；AGY 负责边界清楚的执行步骤。AGY 的退出码
或 `SUCCESS` 只说明调用结束，任务是否完成以主模型检查到的文件、diff 和测试证据为准。

## 触发门

仅在以下条件全部成立时进入编排：

1. 用户明确要求“使用 Antigravity/AGY”，或直接调用 `$agy-orchestrator`。
2. 工作至少包含两个有独立结果和验收条件的步骤。
3. 主模型可以检查 AGY 产生的结果。
4. 委派保持在用户已给出的任务范围和权限内。

若只满足第 1 条，而任务是改一个错字、回答一个问题或完成一次机械操作，由主模型
直接处理。讨论 Antigravity、查询文档或仅提到模型名称不属于执行请求。

## 开始前

1. 读取仓库指令，检查 `git status`、已有 diff、测试入口和目标文件。
2. 保存用户原有改动的基线；后续检查时区分原改动与本次新增改动。
3. 运行 `agy models` 获取实时模型 slug。每次调用都刷新，静态路由表只帮助选型。
4. 将任务拆成顺序步骤。每一步写明目标、允许写入范围、输入契约和验收命令。
5. 默认串行。只有步骤间没有数据或接口依赖，并且允许写入的文件集合完全不相交时，
   才可以并行。

选择工作模型前必须阅读 [模型路由](references/model-routing.md)。准备调用、解析结果、
审查或返工时必须阅读 [编排协议](references/orchestration-protocol.md)。

## 执行闭环

对每个步骤依次执行：

1. 从 `agy models` 的实时结果中选择合适的最低成本模型。
2. 只读分析使用 `--mode plan`；文件实现使用 `--mode accept-edits`。
3. 使用 `--output-format json` 调用 AGY，并保存 `conversation_id`。
4. 调用结束后，主模型亲自检查：
   - JSON `status`、`error`、`denied_actions` 与 AGY 报告；
   - 相对基线新增的文件和完整 diff；
   - 是否超出允许写入范围；
   - 约定的测试、构建或用户可见结果；
   - 用户原有改动是否保持完整。
5. 通过验收后再开始下一步，并把已验收契约作为下一步固定输入。
6. 未通过时使用原 `conversation_id` 发送具体失败证据和不变的验收条件。
7. 同一缺陷最多交给 AGY 修正两轮；之后由主模型接管最小必要修改并重新验证。

`denied_actions` 非空、`response` 为空或产物路径不在步骤合同范围内时，该步骤直接进入
返工或主模型接管，即使 `status` 是 `SUCCESS`。时间压力、token 压力或 AGY 自报成功
都不改变上述验收门。可以减少委派步骤或改由主模型直接完成，但不能省略实时模型检查
和结果验收。

## 交付

所有步骤分别通过后，主模型检查完整 diff 并运行最全面的相关验证。最终回复列出完成
内容、实际验证和仍存在的限制；不得把 AGY 的文字总结当作唯一证据。

## 常见错误

| 情况 | 正确处理 |
|---|---|
| 用户点名 Skill，但任务只有一步 | 主模型直接完成，不启动 AGY |
| 记得某个模型名称 | 仍先运行 `agy models`，只使用实时 slug |
| 多个步骤看起来可以并行 | 同时满足“无依赖”和“写入集合不相交”才并行 |
| JSON 返回 `SUCCESS` | 继续检查真实 diff 和验收命令 |
| JSON 含 `denied_actions` 或空 `response` | 判定步骤未完成，检查部分写入后返工或接管 |
| AGY 报告的文件位于自身 `scratch` | 判定越界，不把该文件当作目标产物 |
| AGY 修正后再次自报完成 | 从头执行该步骤的主模型审查门 |
| 同一问题连续两轮未通过 | 主模型接管最小修改 |
