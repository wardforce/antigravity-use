# AGY 真实探针结果

## 环境

- AGY CLI：1.2.2
- 模型：`gemini-3.8-flash-low`
- 工作目录：`skills/agy-orchestrator-workspace/agy-live-probe`

## 默认 headless 写入

第一次调用返回退出码 0 和 `status: SUCCESS`，但 `response` 为空，并包含：

```json
{"denied_actions":[{"action":"command","display_name":"RunCommand"}]}
```

目标 `result.txt` 没有生成，因此主模型审查判定失败。

## 同会话返工

使用原 `conversation_id` 要求只使用文件工具。返回的 `num_turns` 从 1 增长为 2，但
结果仍为空，并包含被拒绝的 `mcp` 动作。目标文件仍未生成，第二轮审查继续失败。

## AGY 沙箱探针

在 AGY 自身沙箱中运行后，JSON 返回 `SUCCESS`，但 AGY 找不到工作区内的 `task.md`，
并把 `result.txt` 写到了自身 `scratch`。目标工作区仍没有 `result.txt`，因此按越界失败
处理。

## 结论

这组实测证明 `status: SUCCESS` 只可作为传输终态。验收还必须检查
`denied_actions`、空 `response`、目标绝对路径、真实文件和 diff。两轮返工仍未得到
目标产物后，按 Skill 规则由主模型接管。
