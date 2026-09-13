# AGY Orchestrator Skill Design

## Purpose

Create a Codex skill that lets a capable orchestration model delegate long,
decomposable work to Google Antigravity through the `agy` CLI while retaining
responsibility for scope, review, correction, and final verification.

The skill is optimized for Codex sessions using `gpt-5.6-sol` or
`gpt-6-astra`. The user must explicitly ask to use Antigravity/AGY or directly
invoke `$agy-orchestrator`, after which the task must still have multiple
meaningful, independently verifiable steps. Small edits and short questions
remain with the main model.

## Deliverables

The skill will live at `skills/agy-orchestrator/` and contain:

- `SKILL.md`: activation criteria and the orchestration workflow.
- `agents/openai.yaml`: discoverable UI metadata with normal discovery enabled;
  the explicit-intent and complexity predicates in the description prevent
  accidental activation.
- `references/model-routing.md`: the current model inventory, model-specific
  task routing, and runtime refresh rules.
- `references/orchestration-protocol.md`: AGY command patterns, response
  contracts, review gates, revision handling, and failure recovery.

No wrapper program is required for the initial version. The native CLI already
provides the required model discovery, JSON envelope, streaming events, and
conversation resumption. Keeping judgment in the main model also prevents a
script from treating syntactic success as task success.

## Activation

Use the skill when all of the following are true:

1. The user explicitly asks to use Antigravity/AGY in the prompt or directly
   invokes `$agy-orchestrator`.
2. The task can be divided into at least two meaningful steps with explicit
   acceptance checks.
3. Delegation will reduce execution cost or latency without weakening review.
4. The requested work is within the current task's existing scope and
   permissions.

`gpt-5.6-sol` and `gpt-6-astra` are the preferred orchestrator models, not hard
activation requirements. Enter activation checks only when the user expresses
an intent to use Antigravity/AGY or directly invokes the skill. Product
discussion, documentation questions, and model-name mentions are not execution
intent. Keep trivial changes, ambiguous work that first needs user input, and
work whose result the main model cannot inspect with the main model.

## Runtime Discovery and Model Routing

At the start of each invocation, run `agy models`. Its live output is the
authority for usable model slugs. Model names must not be inferred from the
skill's dated snapshot.

The initial routing reference will document every model returned by the local
CLI on 2026-09-13:

1. `gemini-3.8-flash-high`
2. `gemini-3.8-flash-medium`
3. `gemini-3.8-flash-low`
4. `gemini-3.7-flash-high`
5. `gemini-3.7-flash-medium`
6. `gemini-3.7-flash-low`
7. `gemini-3.6-flash-high`
8. `gemini-3.6-flash-medium`
9. `gemini-3.6-flash-low`
10. `gemini-3.1-pro-high`
11. `gemini-3.1-pro-low`
12. `claude-sonnet-4-6`
13. `claude-opus-4-6-thinking`
14. `gpt-oss-120b-medium`

Newly discovered models must be considered using their displayed family,
reasoning level, official characteristics when available, and the task's actual
needs. Removed models must not be selected. The latest suitable Flash model is
the default implementation worker; older Flash families are fallbacks rather
than automatic choices.

## Orchestration Flow

### 1. Establish a baseline

Before delegation, the main model inspects repository instructions, current
status, existing changes, and relevant tests. Existing user changes are not
reset, stashed, or overwritten.

### 2. Decompose the task

The main model creates ordered steps. Each step defines:

- one concrete outcome;
- allowed file scope;
- relevant context and constraints;
- an observable acceptance check;
- dependencies on earlier steps.

Execution is serial by default. Steps may run in parallel only when they have no
dependency on one another and their permitted write sets are disjoint. If file
ownership is uncertain, run them serially.

### 3. Select an AGY model

Choose the least expensive live model that is well matched to the step:

- Latest Flash Low for searches, mechanical edits, formatting, and small tests.
- Latest Flash Medium for ordinary implementation and focused multi-file work.
- Latest Flash High for debugging or implementation requiring deeper reasoning.
- Gemini Pro for architecture, migration, long-context, or difficult planning.
- Claude Sonnet for balanced implementation, review, and documentation.
- Claude Opus for difficult reasoning, long-horizon work, and complex refactors.
- GPT-OSS 120B for text-first reasoning, structured analysis, and an independent
  second opinion.

The detailed routing reference will distinguish every live configuration.

### 4. Execute one step

Use AGY headless mode with an explicit model, working directory, prompt, and
`--output-format json`. Use `--mode accept-edits` for an approved implementation
step and `--mode plan` for read-only analysis. Do not enable
`--dangerously-skip-permissions` by default.

The prompt requires AGY to report:

- the result;
- files inspected and changed;
- validation commands and their results;
- unresolved issues;
- any departure from the assigned scope.

The main model stores the returned `conversation_id` for revisions. It treats a
zero exit code or `status: SUCCESS` as transport success only.

### 5. Review after every step

Before starting the next step, the main model independently verifies:

- the result matches the stated outcome;
- only allowed files changed;
- the diff is technically correct and minimal;
- relevant tests or checks actually pass;
- AGY's summary agrees with the observable workspace state;
- no existing user changes were damaged.

The main model must inspect the artifacts and diff itself. AGY self-review is
supporting evidence, not acceptance.

### 6. Revise or take over

If a step fails review, resume its exact conversation with
`--conversation <id>` and send concrete discrepancies plus the unchanged
acceptance criteria. Review the new result from scratch.

Allow at most two AGY correction rounds for the same defect. If the result is
still unacceptable, or a revision broadens scope, the main model makes the
smallest necessary correction itself and runs the acceptance check.

### 7. Final review

After all steps pass their local gates, the main model performs an integrated
review of the complete diff and runs the broadest relevant verification. Only
then may it report completion to the user.

## AGY Response Handling

The default integration consumes the single JSON envelope emitted by
`--output-format json`:

- `conversation_id`: revision handle.
- `status`: terminal CLI state.
- `response`: worker report.
- `error`: failure detail when present.
- `duration_seconds` and `num_turns`: operational metadata.
- `usage`: token accounting.

`--output-format stream-json` is optional when live tool and step visibility is
valuable. Its terminal `result` event remains the source of the final envelope.

The initial version does not depend on `--json-schema`: local testing found a
provider/tool-schema compatibility failure. It also does not assume
`agy models --output-format json` exists: local CLI 1.2.2 returned a flag error
despite newer online documentation describing machine-readable list output.

## Error Handling

- Unknown model: refresh `agy models`, select another suitable live slug, and
  do not silently use a different family.
- Authentication failure: report the exact condition and preserve all work.
- Non-zero exit or `status: ERROR`: do not review the response as completed
  work; diagnose stderr and the JSON `error` field first.
- Timeout with partial output: inspect workspace state and the warning before
  deciding whether to resume or retry.
- Permission denial: keep the denied action visible and obtain any required
  user approval through the main Codex session.
- Scope violation: reject the step, restore only AGY-owned changes through a
  targeted edit, and preserve pre-existing work.

## Verification Strategy

The completed skill will be checked in four ways:

1. Run the Codex skill validator on the skill directory.
2. Confirm the routing reference contains every live `agy models` entry.
3. Run a small isolated AGY implementation task and inspect its JSON envelope,
   file diff, and test result.
4. Send a deliberate reviewer correction through `--conversation` and verify
   the revised artifact before final acceptance.

## Sources

- Google Antigravity CLI headless mode:
  https://antigravity.google/docs/cli/headless/
- Google Antigravity models:
  https://antigravity.google/docs/models/
- Google Antigravity CLI features:
  https://antigravity.google/docs/cli/features/
- Gemini 3.8 Flash announcement:
  https://antigravity.google/blog/gemini-3-8-flash-in-google-antigravity
- Gemini 3.1 Pro announcement:
  https://antigravity.google/blog/gemini-3-1-pro-in-google-antigravity
- Anthropic model overview:
  https://docs.anthropic.com/en/docs/about-claude/models
- OpenAI GPT-OSS 120B model page:
  https://developers.openai.com/api/docs/models/gpt-oss-120b
