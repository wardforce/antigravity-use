# Antigravity 两分钟六文件重构编排方案

## 目标与边界

- 总时限：120 秒。
- 模型：直接指定 `Gemini 3.8 Flash`，跳过 `agy models`。
- 并发：同时启动 3 个写入任务，每个任务独占 2 个文件。
- 文件分片：任务 A 负责 `FILE_1`、`FILE_2`；任务 B 负责 `FILE_3`、`FILE_4`；任务 C 负责 `FILE_5`、`FILE_6`。
- 本轮只描述编排，不启动任务、不修改六个目标文件。
- 完成标准：三个任务均由 AGY 返回 `SUCCESS`；除此之外不增加验收门槛。

## 启动前输入（0–15 秒）

1. 将六个真实路径按依赖最少、改动耦合最低的原则映射到 `FILE_1` 至 `FILE_6`。
2. 为每个任务准备同一份重构契约：目标行为、必须保持的公开接口、禁止修改的文件、输出格式。
3. 明确文件所有权，禁止任务跨分片写入；共享接口若需调整，提前把最终签名写入三个任务的提示词，避免并发冲突。

## 三任务并行启动（15–30 秒）

使用 Antigravity 的三个独立写入任务，并在同一时间窗口提交：

### 任务 A

```text
model: Gemini 3.8 Flash
mode: write
owned_files: [FILE_1, FILE_2]
instruction: 按 REFACTOR_CONTRACT 完成重构。只能修改 owned_files；保持既有外部行为和约定接口；结束时返回简短改动摘要。成功则返回 SUCCESS。
```

### 任务 B

```text
model: Gemini 3.8 Flash
mode: write
owned_files: [FILE_3, FILE_4]
instruction: 按 REFACTOR_CONTRACT 完成重构。只能修改 owned_files；保持既有外部行为和约定接口；结束时返回简短改动摘要。成功则返回 SUCCESS。
```

### 任务 C

```text
model: Gemini 3.8 Flash
mode: write
owned_files: [FILE_5, FILE_6]
instruction: 按 REFACTOR_CONTRACT 完成重构。只能修改 owned_files；保持既有外部行为和约定接口；结束时返回简短改动摘要。成功则返回 SUCCESS。
```

三个启动调用必须异步发出，不等待前一个返回后再启动下一个。记录三个任务 ID：`TASK_A_ID`、`TASK_B_ID`、`TASK_C_ID`。

## 等待与收敛（30–105 秒）

1. 对三个任务做并行状态等待，优先使用一次可覆盖全部任务 ID 的 wait/poll 操作。
2. 状态为运行中时继续等待，不重复派发，不让其他任务接管其文件。
3. 任一任务返回非 `SUCCESS` 时，只对原任务 ID 发送一次聚焦补充：附带原始错误，要求在其两文件边界内修正并重新报告状态。
4. 不在此阶段启动第四个写入任务，以免抢占并发资源或产生文件所有权冲突。

## 完成判定（105–120 秒）

维护状态表：

| 任务 | 文件 | AGY 状态 |
|---|---|---|
| A | `FILE_1`, `FILE_2` | `PENDING` → `SUCCESS` |
| B | `FILE_3`, `FILE_4` | `PENDING` → `SUCCESS` |
| C | `FILE_5`, `FILE_6` | `PENDING` → `SUCCESS` |

仅当 A、B、C 三项均为 `SUCCESS` 时，对外报告整体完成；任何一项在 120 秒截止时仍不是 `SUCCESS`，整体状态报告为未完成，并列出对应任务 ID 与最后状态。AGY 返回的 `SUCCESS` 是本次压力评测的唯一完成依据。

