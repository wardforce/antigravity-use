# AGY Orchestrator Skill Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 创建一个由 Codex 高阶模型编排、通过 `agy` 委派长流程任务、逐步审查并处理返工的中文 Skill。

**Architecture:** `SKILL.md` 只保留触发判断和核心闭环；模型的完整实时路由放进 `references/model-routing.md`；CLI 调用、JSON 返回和逐步验收协议放进 `references/orchestration-protocol.md`。运行时先调用 `agy models`，因此静态快照只提供分工参考，不决定模型是否可用。

**Tech Stack:** Codex Agent Skills Markdown、YAML、Google Antigravity CLI 1.2.2、PowerShell、Git。

**Spec:** `docs/superpowers/specs/2026-09-13-agy-orchestrator-design.zh-CN.md`

## Global Constraints

- Skill 正文和参考文档使用中文；命令、模型 slug、JSON 字段保持原文。
- 仅在用户明确要求使用 Antigravity/AGY 或直接调用 `$agy-orchestrator` 时进入触发判断。
- 显式触发后，任务仍须至少含两个可验收步骤。
- 默认串行；只有无依赖且写入文件集合完全不相交时允许并行。
- 每个 AGY 步骤后由主模型检查产物、diff 和测试，再决定通过、返工或接管。
- 每个缺陷最多交给同一 AGY 会话返工两轮。
- 每次使用时以 `agy models` 的实时输出为准。

---

### Task 1: RED 基线测试

**Files:**
- Create: `skills/agy-orchestrator/evals/evals.json`
- Create: `skills/agy-orchestrator-workspace/iteration-1/*/without_skill/outputs/result.md`

**Interfaces:**
- Consumes: 已批准的中文设计规格。
- Produces: 三个无 Skill 基线行为样本和可验证失败模式。

- [x] **Step 1:** 写覆盖复杂委派、简单误触发、并行冲突和时间压力的测试提示词。
- [x] **Step 2:** 使用独立代理在无 Skill 条件下执行测试。
- [x] **Step 3:** 记录复杂度门槛、实时模型发现、逐步审查、会话返工和并行边界表现。
- [x] **Step 4:** 只根据观察到的压力用例失败决定正式 Skill 的重点约束。

### Task 2: GREEN 编写最小 Skill

**Files:**
- Create: `skills/agy-orchestrator/SKILL.md`
- Create: `skills/agy-orchestrator/agents/openai.yaml`
- Create: `skills/agy-orchestrator/references/model-routing.md`
- Create: `skills/agy-orchestrator/references/orchestration-protocol.md`

**Interfaces:**
- Consumes: Task 1 的失败模式和 `agy models` 实时清单。
- Produces: 可被 Codex 发现、读取和执行的 `agy-orchestrator` Skill。

- [x] **Step 1:** 创建最小目录结构和 UI 元数据。
- [x] **Step 2:** 写 `SKILL.md`，包含触发逻辑、串行默认值和逐步审查闭环。
- [x] **Step 3:** 写逐项模型路由表，覆盖实时列表中的所有模型。
- [x] **Step 4:** 写 CLI 协议，包含 JSON、stream-json、conversation 返工和 Windows 注意事项。
- [x] **Step 5:** 运行两套 `quick_validate.py`，结果均为 `Skill is valid!`。

### Task 3: REFACTOR 行为复测

**Files:**
- Create: `skills/agy-orchestrator-workspace/iteration-1/*/with_skill/outputs/result.md`
- Modify: `skills/agy-orchestrator/SKILL.md`（仅当复测暴露真实缺口）
- Modify: `skills/agy-orchestrator/references/*.md`（仅当复测暴露真实缺口）

**Interfaces:**
- Consumes: Task 2 的完整 Skill。
- Produces: 同一测试集下的合规结果和缺口修正。

- [x] **Step 1:** 用带 Skill 的独立代理运行同一组测试。
- [x] **Step 2:** 检查是否正确拒绝简单任务委派、是否动态枚举模型、是否逐步审查。
- [x] **Step 3:** 检查并行仅用于无依赖且写入范围不相交的步骤。
- [x] **Step 4:** 针对真实 `denied_actions` 和 scratch 路径问题做最小修正并重测。

### Task 4: 真实 AGY 闭环与交付

**Files:**
- Create: `skills/agy-orchestrator-workspace/agy-live-probe/*`
- Modify: `skills/agy-orchestrator/evals/evals.json`（加入最终断言）

**Interfaces:**
- Consumes: 通过行为复测的 Skill。
- Produces: CLI、模型覆盖、返工和最终审查的可观察证据。

- [x] **Step 1:** 在隔离目录运行最小 AGY 文件生成任务，捕获 JSON 信封和权限拒绝。
- [x] **Step 2:** 主模型检查目标文件，然后通过原 `conversation_id` 请求定向返工。
- [x] **Step 3:** 验证 `num_turns` 增长、目标文件仍缺失及沙箱 scratch 路径偏移。
- [x] **Step 4:** 对 Skill 运行最终校验、模型覆盖检查和 `git diff --check`。
- [x] **Step 5:** 提交 Skill、测试定义和验证结果。
