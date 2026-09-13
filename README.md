# Antigravity Use (`antigravity-use`)

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![AGY CLI](https://img.shields.io/badge/AGY%20CLI-1.2.2+-brightgreen.svg)](https://github.com)

English | [简体中文](README_ZH_CN.md)

A robust orchestration framework and agent skill suite for delegating multi-step development tasks from primary orchestrator models (such as OpenAI Codex `gpt-5.6-sol`, `gpt-6-astra`, Claude, etc.) to **Google Antigravity (`agy` CLI)**, ensuring strict verification gates, bounded permissions, and controlled rework loops.

---

## Overview

Delegating long-horizon software engineering tasks to autonomous CLI agents often risks context drifting, scope creep, silent failures, and unverified diffs. 

`antigravity-use` implements the **AGY Orchestrator** skill (`skills/agy-orchestrator`), enforcing a supervisory architecture:
- **The Primary Orchestrator** plans, sets boundaries, writes contracts, and independently verifies diffs, test passes, and runtime results.
- **The AGY CLI Worker** executes bounded, single-outcome steps within an explicitly permitted read/write file scope.
- **Strict Verification Gate**: AGY's `SUCCESS` exit code or status only indicates CLI completion; task completion is strictly judged by independent file and test inspection.

---

## Key Features

1. **Strict Gatekeeping & Contract-Driven Execution**
   - Requires explicit user intent (e.g., `$agy-orchestrator` or asking to use AGY) and at least 2 independently verifiable steps.
   - Each step operates under an explicit contract: goal, context, allowed read/write paths, invariants, and acceptance checks.
2. **Runtime Dynamic Model Routing**
   - Queries `agy models` dynamically at runtime to select the most cost-effective live model.
   - Intelligent tiering across Gemini Flash (mechanical edits / debugging), Gemini Pro (architecture / planning), Claude Sonnet / Opus (reasoning / refactors), and GPT-OSS.
3. **Robust Protocol & Rework Recovery**
   - Headless invocation via JSON envelope (`--output-format json`).
   - Tracks `conversation_id` for targeted multi-turn rework (capped at 2 rework turns before orchestrator takeover).
   - Treats `denied_actions`, empty responses, or out-of-scope files as immediate rejection triggers.
4. **Extension & Capability Discovery**
   - Discovers project-level and global MCP tools, Skills, Plugins, Hooks, and Rules (`references/agy-extensions.md`).
   - Prevents capability leaks while safely sharing needed tools.
5. **Safe Serial-by-Default Concurrency**
   - Defaults to serial execution. Concurrency is only permitted when steps have zero dependency and mutually disjoint file write sets.

---

## Directory Structure

```text
antigravity-use/
├── README.md                                  # English documentation
├── README_ZH_CN.md                            # Chinese documentation (简体中文)
├── docs/
│   └── superpowers/
│       ├── specs/
│       │   ├── 2026-09-13-agy-orchestrator-design.md        # English design spec
│       │   └── 2026-09-13-agy-orchestrator-design.zh-CN.md  # Chinese design spec
│       └── plans/
│           └── 2026-09-13-agy-orchestrator.md               # Implementation plan & tracking
└── skills/
    ├── agy-orchestrator/                      # Core AGY Orchestrator skill
    │   ├── SKILL.md                           # Skill definition & execution loop
    │   ├── agents/openai.yaml                 # OpenAI / Codex skill metadata
    │   ├── evals/evals.json                   # Evaluation benchmark cases
    │   └── references/
    │       ├── model-routing.md               # Dynamic model routing catalog
    │       ├── orchestration-protocol.md      # Headless CLI invocation & review protocol
    │       └── agy-extensions.md              # MCP / Skills / Plugins discovery rules
    └── agy-orchestrator-workspace/            # Test runs, probe logs, and evaluation iterations
```

---

## Quick Start

### 1. Prerequisites

- Installed and authenticated `agy` CLI:
  ```powershell
  agy --version
  agy models
  ```

### 2. Basic Invocation Protocol

#### Read-Only Analysis (`--mode plan`)
```powershell
agy --model <LIVE_MODEL_SLUG> --mode plan --output-format json --print "<STEP_PROMPT>"
```

#### Implementation Step (`--mode accept-edits`)
```powershell
agy --model <LIVE_MODEL_SLUG> --mode accept-edits --output-format json --print "<STEP_PROMPT>"
```

#### Rework via Resumed Conversation
```powershell
agy --model <LIVE_MODEL_SLUG> --conversation-id <CONVERSATION_ID> --mode accept-edits --output-format json --print "<REWORK_FEEDBACK>"
```

---

## Verification & Evals

The skill includes benchmark and regression test suites under `skills/agy-orchestrator-workspace/`:
- **Denied actions**: Validates handling when headless calls encounter blocked file operations.
- **Scratch path isolation**: Ensures worker artifacts in sandbox paths are not mistaken for project deliverables.
- **Parallel conflict prevention**: Guarantees serial execution when write scopes overlap.

---

## Documentation

- [Design Specification (English)](docs/superpowers/specs/2026-09-13-agy-orchestrator-design.md)
- [Design Specification (中文)](docs/superpowers/specs/2026-09-13-agy-orchestrator-design.zh-CN.md)
- [Orchestrator Protocol](skills/agy-orchestrator/references/orchestration-protocol.md)
- [Model Routing Guide](skills/agy-orchestrator/references/model-routing.md)
- [AGY Extension Discovery](skills/agy-orchestrator/references/agy-extensions.md)
