# Antigravity Use (`antigravity-use`)

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![AGY CLI](https://img.shields.io/badge/AGY%20CLI-1.2.2+-brightgreen.svg)](https://github.com)

[English](README.md) | 简体中文

面向高阶编排模型（如 OpenAI Codex `gpt-5.6-sol`、`gpt-6-astra`、Claude 等）的 **Google Antigravity (`agy` CLI)** 多步骤任务编排框架与 Agent Skill 套件。提供严密的验收门禁、范围限制与返工收敛闭环。

---

## 项目概述

将长流程工程任务直接委派给自主命令行智能体时，往往面临上下文漂移、越界修改、静默失败与未经测试验收等风险。

`antigravity-use` 实现了 **AGY 编排器** 技能（`skills/agy-orchestrator`），确立主从协同原则：
- **主编排模型**：负责任务拆分、制定范围边界、起草步骤合同，并亲自独立检查文件 diff、测试用例与运行结果。
- **AGY CLI 执行器**：在明确划定的允许读写路径内，负责边界清晰的单一成果步骤。
- **严格验收门禁**：AGY 的退出码或 `SUCCESS` 仅代表命令行进程结束；任务是否完成，严格以主模型在工作区观察到的真实变更与测试证据为准。

---

## 核心特性

1. **触发门禁与合同化执行**
   - 仅在用户明确表达委派意图（如调用 `$agy-orchestrator` 或显式指明使用 AGY）且任务包含至少 2 个独立可验收步骤时触发。
   - 每一步均具备明确的提示词合同：包含目标、上下文、允许读写路径、保持不变范围与验收命令。
2. **运行时动态模型路由**
   - 每次调用均先运行 `agy models` 获取实时模型列表，按任务难度选择最低成本可用模型。
   - 涵盖 Gemini Flash（机械修改、测试运行、初级实现）、Gemini Pro（架构迁移、长上下文）、Claude Sonnet/Opus（复杂重构、高难度逻辑）及开源模型。
3. **协议化调用与会话返工**
   - 采用 Headless JSON 封装格式（`--output-format json`）。
   - 保留 `conversation_id` 进行定向返工，单个缺陷限制在 2 轮内修复，超限后由主模型接管以防止死循环。
   - 严格识别 `denied_actions`、空 response 与沙箱 `scratch` 越界产物，坚决触发返工或人工干预。
4. **扩展能力发现与安全隔离**
   - 自动梳理项目级与全局级 MCP、Skills、Plugins、Hooks 与 Rules（参见 `references/agy-extensions.md`）。
   - 仅将步骤相关且已生效能力注入上下文，严格防止密钥泄露与无权限改动。
5. **默认串行与独立并行机制**
   - 步骤间默认严格串行。仅当多个步骤无前后数据依赖且写入文件路径完全不相交时，才允许并行执行。

---

## 目录结构

```text
antigravity-use/
├── README.md                                  # 英文说明文档
├── README_ZH_CN.md                            # 中文说明文档
├── docs/
│   └── superpowers/
│       ├── specs/
│       │   ├── 2026-09-13-agy-orchestrator-design.md        # 英文设计规格
│       │   └── 2026-09-13-agy-orchestrator-design.zh-CN.md  # 中文设计规格
│       └── plans/
│           └── 2026-09-13-agy-orchestrator.md               # 落地执行计划与实施检查
└── skills/
    ├── agy-orchestrator/                      # 核心 AGY 编排器 Skill
    │   ├── SKILL.md                           # Skill 定义与编排闭环规范
    │   ├── agents/openai.yaml                 # OpenAI / Codex 元数据配置
    │   ├── evals/evals.json                   # 评估测试集
    │   └── references/
    │       ├── model-routing.md               # 动态模型选型与路由说明
    │       ├── orchestration-protocol.md      # 命令行协议、审查门与返工规则
    │       └── agy-extensions.md              # 扩展发现（MCP / Skill / Rule 等）
    └── agy-orchestrator-workspace/            # 评测基线、实测结果与探针记录
```

---

## 快速上手

### 1. 环境准备

确保本地已安装并登录 `agy` CLI：
```powershell
agy --version
agy models
```

### 2. 核心调用命令

#### 只读分析（`--mode plan`）
```powershell
agy --model <LIVE_MODEL_SLUG> --mode plan --output-format json --print "<STEP_PROMPT>"
```

#### 代码修改与实现（`--mode accept-edits`）
```powershell
agy --model <LIVE_MODEL_SLUG> --mode accept-edits --output-format json --print "<STEP_PROMPT>"
```

#### 在原有会话中定向返工
```powershell
agy --model <LIVE_MODEL_SLUG> --conversation-id <CONVERSATION_ID> --mode accept-edits --output-format json --print "<REWORK_FEEDBACK>"
```

---

## 评测与测试套件

在 `skills/agy-orchestrator-workspace/` 中提供了完整的基准评测用例：
- **操作被拒（Denied Actions）**：验证无权限写操作触发时的错误拦截与收敛能力。
- **沙箱路径隔离（Scratch Path Isolation）**：防止将临时目录文件误判为主工程产物。
- **并行冲突防护（Parallel Conflict Prevention）**：验证写入范围重叠时自动保持串行的控制机制。

---

## 详细参考文档

- [中文设计规格 (Design Spec)](docs/superpowers/specs/2026-09-13-agy-orchestrator-design.zh-CN.md)
- [英文设计规格 (Design Spec EN)](docs/superpowers/specs/2026-09-13-agy-orchestrator-design.md)
- [编排协议手册 (Protocol)](skills/agy-orchestrator/references/orchestration-protocol.md)
- [模型路由指南 (Model Routing)](skills/agy-orchestrator/references/model-routing.md)
- [扩展能力发现规则 (AGY Extensions)](skills/agy-orchestrator/references/agy-extensions.md)
