# AGY 模型路由

## 选择顺序

每次使用时先运行：

```powershell
agy models
```

只从该命令当前返回的 slug 中选择模型。下面是 2026-09-13 在本机 AGY CLI 1.2.2
中实测到的完整快照，用于说明分工，不代表未来仍可用。

优先选择足以完成步骤的最低成本模型。默认使用最新 Flash 家族；旧版 Flash 用于用户
指定、回归对比、配额或兼容回退。切换模型家族时在执行记录中说明原因。

## 当前 14 个配置

| 实时 slug 快照 | 特点 | 适合的步骤 |
|---|---|---|
| `gemini-3.8-flash-high` | 当前最新 Flash 的高推理档，兼顾速度和较深推理 | 困难调试、跨文件实现、复杂测试修复 |
| `gemini-3.8-flash-medium` | 当前最新 Flash 的平衡档 | 默认日常编码、多文件但边界清楚的实现 |
| `gemini-3.8-flash-low` | 当前最新 Flash 的低延迟档 | 搜索、机械替换、格式整理、小型测试 |
| `gemini-3.7-flash-high` | 上一代 Flash 高推理档，官方说明仍适合强调计算效率的流程 | 需要较深推理但希望复现 3.7 行为的任务 |
| `gemini-3.7-flash-medium` | 上一代 Flash 平衡档 | 3.8 不可用时的普通编码回退 |
| `gemini-3.7-flash-low` | 上一代 Flash 低推理档 | 3.8 不可用时的机械任务回退 |
| `gemini-3.6-flash-high` | 较早 Flash 高推理档 | 版本回归、兼容验证或用户明确指定 |
| `gemini-3.6-flash-medium` | 较早 Flash 平衡档 | 固定旧行为的普通实现或对照测试 |
| `gemini-3.6-flash-low` | 较早 Flash 低推理档 | 固定旧行为的轻量机械步骤 |
| `gemini-3.1-pro-high` | Pro 高推理档，官方强调复杂规划、长流程和跨代码库上下文 | 架构、迁移方案、复杂根因分析、长上下文任务 |
| `gemini-3.1-pro-low` | Pro 的较低推理档 | 需要 Pro 家族能力但范围较窄的规划或审查 |
| `claude-sonnet-4-6` | Thinking 配置，平衡速度、智能和日常代理工作 | 代码实现、代码审查、文档、长上下文综合任务 |
| `claude-opus-4-6-thinking` | 当前 AGY 提供的 Claude 深推理配置 | 困难推理、复杂重构、疑难故障和长流程审查 |
| `gpt-oss-120b-medium` | 文本输入输出、可调推理且具代理和结构化输出能力 | 文本型推理、结构化分析、独立第二意见 |

## 新模型或缺失模型

- 实时列表出现新模型：先查其官方说明；若当前任务不需要联网调研，则根据显示的家族、
  推理档和 AGY 描述做保守选择，并注明依据。
- 快照中的模型未出现在实时列表：从候选中移除，不尝试猜测别名。
- 指定 slug 报 unknown model：重新运行 `agy models`，再选择一个满足任务要求的实时
  slug；不得静默降级到不同家族。
- 多个模型都合适：先选较低推理档；验收失败且问题确实属于推理不足时，再升级模型。

## 官方依据

- Antigravity 当前模型：https://antigravity.google/docs/models/
- Gemini 3.8 Flash：https://antigravity.google/blog/gemini-3-8-flash-in-google-antigravity
- Gemini 3.1 Pro：https://antigravity.google/blog/gemini-3-1-pro-in-google-antigravity
- Claude 模型总览：https://docs.anthropic.com/en/docs/about-claude/models
- GPT-OSS 120B：https://developers.openai.com/api/docs/models/gpt-oss-120b

