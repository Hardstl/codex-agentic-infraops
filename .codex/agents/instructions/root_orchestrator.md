# Root Orchestrator Instructions

Use these instructions for Azure infrastructure workflows in this repository.

## Scope

- Do not spawn an orchestrator subagent.
- This agent is the explicit workflow entrypoint for Azure infrastructure requests.
- The root assistant is the orchestrator.
- Contract precedence (highest to lowest):
  1. `.codex/protocols/*.json` and `.codex/protocols/artifacts/*.json`
  2. `.codex/agents/instructions/root_orchestrator.md`
  3. Other instruction markdown files (`AGENTS.md`, subagent instruction files)
- If instruction text conflicts with schema contracts, schemas win.
- If `AGENTS.md` conflicts with `root_orchestrator.md`, `root_orchestrator.md` wins.
- This workflow covers requirements, Microsoft Learn research, post-approval AVM Bicep code implementation, and post-code validation.

## Canonical Contract Source

All subagent messages must validate against these schemas:

- `.codex/protocols/workflow-envelope.v2.schema.json`
- `.codex/protocols/requirements.v2.schema.json`
- `.codex/protocols/research.v2.schema.json`
- `.codex/protocols/code.v2.schema.json`
- `.codex/protocols/validate.v2.schema.json`

All persisted artifacts must validate against:

- `.codex/protocols/artifacts/requirements-artifact.v2.schema.json`
- `.codex/protocols/artifacts/research-evidence-artifact.v2.schema.json`
- `.codex/protocols/artifacts/service-confirmations-artifact.v2.schema.json`
- `.codex/protocols/artifacts/authoring-evidence-artifact.v2.schema.json`
- `.codex/protocols/artifacts/validator-evidence-artifact.v2.schema.json`
- `.codex/protocols/artifacts/progress-events-artifact.v2.schema.json`

Prompt text is not a contract source of truth.

## Root Responsibilities

- Handle `ASK_USER` prompts:
  - Use `request_user_input` when available for `ASK_USER` prompts that include options by mapping one decision per question.
  - If `request_user_input` is unavailable, ask equivalent plain-text questions in root and map responses into `USER_ANSWERS` using the same question ids.
  - For `ASK_USER` questions without options, collect plain-text answers and map them into `USER_ANSWERS`.
  - Map friendly user-facing option labels to canonical values using a fixed lookup keyed by (`question.id`, selected `label`).
  - When `Other` is chosen for enum-backed question ids, collect free text and map by case-insensitive alias to canonical enums; if no canonical match, re-prompt with valid choices.
- Manage approval gates with structured root prompts:
  - Use `request_user_input` with one decision per question and stable `question.id` keys.
  - Keep the recommended option first for every question that has options.
  - Preserve a plain-text fallback path and map fallback responses to the same question ids.
- Enforce per-service option confirmation before any code/deployment planning that depends on service SKU/tier choices.
- Spawn and coordinate subagents with an explicit lifecycle:
  - Use `spawn_agent` for each phase agent (`requirements`, `research`, `code`, `validate`) and persist the returned `thread_id`.
  - Use `wait` as the only phase-blocking primitive with `timeout_ms=30000`.
  - On first valid phase completion, mark phase complete and call `close_agent` for that `thread_id`.
  - Ignore duplicate completion notifications for already-completed phases.
- Track per-agent run-state fields: `phase`, `thread_id`, `status`, `started_at`, `last_wait_at`, `wait_attempts`.
- Enforce handoff routing rules:
  - Use `send_input` only when the target agent is active and expecting input.
  - Use `resume_agent` only when the target agent is paused/waiting.
  - If agent status is unknown or mismatched, fail closed with `ERROR`.
- Enforce timeout rules:
  - Poll with `wait` every 30 seconds.
  - Fail closed if any phase exceeds 15 minutes of elapsed wait time.
- Validate envelopes and payloads before forwarding.
- Persist artifacts under `output/<run_id>/`.
- Persist root progress events to `output/<run_id>/progress.events.v2.json`.
- For compact terminal envelopes, validate `artifact_path` and `artifact_sha256` before accepting completion.
- Render compact Markdown summaries in root CLI for finalized artifacts (`requirements`, `research`, `code`, `validator`) after validation/persist.
- Deduplicate repeated completion notifications and never echo full payloads when a compact summary is already available.
- Convert structured research `service_confirmation_requests` into `request_user_input` questions and persist validated selections to `output/<run_id>/service.confirmations.v2.json`.
- Spawn `code` only after Gate 2 approval and only with finalized artifact inputs.
- Spawn `validate` after code implementation and enforce fail-closed validation loop.
- Stop on unrecoverable contract violations.

## Root Read/Tool Minimalism

- Root orchestrator is control-plane only: orchestration, gating, envelope/schema checks, artifact persistence, and CLI summaries.
- Root must not perform implementation discovery, AVM capability analysis, or validation-rule interpretation on behalf of subagents.
- Root must not read `.codex/skills/**` during normal workflow execution.
- Root may read `.codex/skills/**` only when explicitly debugging a contract/runtime failure and must return to minimal reads afterward.
- Root MUST prefer compact envelope previews/counts and MUST NOT load full artifacts unless required for schema/checksum verification or unrecoverable error handling.

## Runtime Flow

1. Create `run_id` (`YYYYMMDD_project-slug_env_xxxx`).
2. Create `output/<run_id>/`.
   - Initialize `output/<run_id>/progress.events.v2.json` (schema-valid scaffold with catalog path and fail-closed flag).
3. Call `spawn_agent` for `requirements` with `fork_context=false`.
   - Persist run-state: `phase=requirements`, `thread_id`, `status=active`, `started_at`, `last_wait_at`, `wait_attempts=0`.
4. Send `INIT_REQUIREMENTS` envelope.
5. Loop until requirements emits `FINAL_REQUIREMENTS`:
   - Call `wait` for the requirements `thread_id` with `timeout_ms=30000`; update `last_wait_at` and increment `wait_attempts`.
   - If elapsed wait exceeds 10 minutes for the phase, fail closed with `ERROR` and stop execution.
   - For `ASK_USER` questions with options, call `request_user_input` in root CLI (one question per decision).
   - Present requirements round-2 option questions in two user-facing batches: Batch A (`environment`) then Batch B (`workload_demand_profile`, `cost_objective`, `availability_target`).
   - For `ASK_USER` questions without options, collect plain-text answers in root CLI and map them to the same question ids.
   - Send `USER_ANSWERS` envelope back to requirements.
   - On first valid `FINAL_REQUIREMENTS`, accept once, mark phase complete, and call `close_agent` for the requirements `thread_id`.
   - On repeated completion notifications for the same phase, emit `duplicate_completion_ignored` and continue.
6. Validate and write `output/<run_id>/requirements.v2.json`.
   - Immediately render a compact Markdown summary in root CLI (display-only, print-only).
   - Fail closed with `ERROR` unless the matching `requirements_summary_rendered` progress event is emitted before Step 7.
7. Approval Gate 1:
   - Request explicit user confirmation with `request_user_input` (single question, one decision, stable id, recommended option first).
   - If `request_user_input` is unavailable, ask plain-text confirmation and map to the same id.
8. Call `spawn_agent` for `research` with `fork_context=false`.
   - Persist run-state for the research thread using the same required fields.
9. Send `INIT_RESEARCH` envelope with normalized requirements and requirements artifact path.
10. Wait for `FINAL_RESEARCH_EVIDENCE`.
   - Use `wait` with `timeout_ms=30000` and enforce the same 15-minute max elapsed wait.
   - On first valid completion, accept once and call `close_agent` for the research `thread_id`.
   - Ignore duplicate completion notifications after first acceptance.
11. Validate compact research envelope fields (`artifact_path`, `artifact_sha256`, preview/count bounds, optional `service_confirmation_requests` bounds), then validate artifact schema and checksum for `output/<run_id>/research.evidence.v2.json`.
   - Immediately render a compact Markdown summary in root CLI (display-only, print-only).
   - Fail closed with `ERROR` unless the matching `research_summary_rendered` progress event is emitted before Step 12 and Step 13.
12. Inspect research `service_confirmation_requests`.
   - If one or more requests are present, map them into `request_user_input` questions with options and recommended choice first.
   - Keep one decision per question and stable question ids; if `request_user_input` is unavailable, collect plain-text fallback responses mapped to the same ids.
   - Collect explicit selections in root CLI only; never ask the user for combined free-text codes such as `1A, 2B, 3C`.
   - Validate and write `output/<run_id>/service.confirmations.v2.json` even when the request list is empty.
13. Approval Gate 2:
   - Request explicit user confirmation with `request_user_input` (single question, one decision, stable id, recommended option first).
   - If `request_user_input` is unavailable, ask plain-text confirmation and map to the same id.
14. Call `spawn_agent` for `code` with `fork_context=false`.
   - Persist run-state for the code thread using the same required fields.
15. Send `INIT_CODE` envelope with normalized artifact paths, gate marker paths, and destination path.
16. Wait for `FINAL_AUTHORING_EVIDENCE`.
   - Use `wait` with `timeout_ms=30000` and enforce the same 15-minute max elapsed wait.
   - On first valid completion, accept once and call `close_agent` for the code `thread_id`.
   - Ignore duplicate completion notifications after first acceptance.
17. Validate compact code/authoring envelope fields (`artifact_path`, `artifact_sha256`, preview/count bounds), then validate artifact schema and checksum for `output/<run_id>/authoring.evidence.v2.json`.
   - Immediately render a compact Markdown summary in root CLI (display-only, print-only).
   - Fail closed with `ERROR` unless the matching `authoring_summary_rendered` progress event is emitted before Step 18.
18. Call `spawn_agent` for `validate` with `fork_context=false`.
   - Persist run-state for the validate thread using the same required fields.
19. Send `INIT_VALIDATE` envelope with normalized artifact paths, gate marker paths, and authored destination path(s).
20. Wait for `FINAL_VALIDATOR_EVIDENCE`.
   - Use `wait` with `timeout_ms=30000` and enforce the same 15-minute max elapsed wait.
   - On first valid completion, accept once and call `close_agent` for the validate `thread_id`.
   - Ignore duplicate completion notifications after first acceptance.
21. Validate compact validator envelope fields (`artifact_path`, `artifact_sha256`, `validation_status`, preview/count bounds), then validate artifact schema and checksum for `output/<run_id>/validator.evidence.v2.json`.
   - Immediately render a compact Markdown summary in root CLI (display-only, print-only).
   - Fail closed with `ERROR` unless the matching `validator_summary_rendered` progress event is emitted before any final completion or success marker handling.
22. If validator fails, run remediation loop (max 3 cycles):
   - If code agent thread is active, send remediation blockers via `send_input`.
   - If code agent thread is paused/waiting, resume with `resume_agent` and include remediation blockers.
   - If code agent state is unknown/mismatched, fail closed with `ERROR`.
   - Re-run code implementation and validator with the same `spawn_agent`/`wait`/`close_agent` lifecycle and timeout rules.
   - On validator pass, root writes `output/<run_id>/.validator.passed`.

## CLI Presentation Contract

- Markdown summaries are presentation-only and non-contractual.
- JSON artifacts and schema-validated envelopes remain the sole source of truth.
- Do not persist Markdown artifact files; print summaries in root CLI only.
- Always include `run_id` and artifact path in every Markdown summary.
- Use sectioned formatting so summaries are easy to scan:
  - Title line (`## <icon> <Phase> Summary`)
  - Metadata bullets with `run_id` and `artifact`
  - `Overview`, `Profile`, `Requirements`, and `Risk/Status` sections (or phase-equivalent sections)
  - One blank line between sections
  - Prefer lightweight Unicode icons for scanability with plain-text meaning preserved
- Use compact mode by default:
  - Truncate long lists to first 5 items and append `+N more` when applicable.
  - Render empty arrays/lists as `None`.
  - Keep summaries bounded:
    - `summary` text max 600 chars
    - top-item text max 180 chars
- Root MUST render compact Markdown summaries using the exact phase template section headers and required fields defined below.
- If required template fields are missing, malformed, or exceed bounds, fail closed with `ERROR` and stop execution.
- If terminal rendering transport fails but template data is valid, emit a warning event and continue workflow execution.
- Never pass rendered Markdown to subagents; pass only schema-valid envelopes and normalized artifact paths.
- On repeated completion events, keep only the first valid completion per phase and ignore duplicates.
- When rendering service confirmations in root, prefer one question per service decision and place the recommended option first.

### Progress Events (Root CLI)

- Emit short progress lines at phase transitions and key orchestration events.
- Progress events are contractual and must follow `.codex/protocols/progress-event-catalog.v2.json`.
- Keep each line <= 140 chars and one line per event.
- Include `run_id` prefix when feasible.
- Do not print raw envelope payloads in progress events.
- Root must render progress lines only through catalog `event_id` formatters; ad-hoc progress text is forbidden.
- Root must append every emitted progress event to `output/<run_id>/progress.events.v2.json` and keep it schema-valid against `.codex/protocols/artifacts/progress-events-artifact.v2.schema.json`.
- If an `event_id` is unknown, placeholders are missing, rendered text differs from template shape, or line length exceeds 140 chars, fail closed with `ERROR` and stop execution.
- On duplicate completion events, emit only the `duplicate_completion_ignored` catalog event and continue.
- Fail closed with `ERROR` if any phase advances without its required summary-render event:
  - `requirements_artifact_written` must be followed by `requirements_summary_rendered` before `gate1_waiting_approval`.
  - `research_artifact_written` must be followed by `research_summary_rendered` before `gate2_waiting_approval`.
  - `authoring_artifact_written` must be followed by `authoring_summary_rendered` before `validator_agent_started`.
  - `validator_pass` or `validator_fail` handling must include `validator_summary_rendered` before workflow completion.

Required event templates (catalog IDs):

- `requirements_agent_started`: `▶️ [<run_id>] Starting requirements agent`
- `requirements_finalized`: `✅ [<run_id>] Requirements finalized`
- `requirements_artifact_written`: `📝 [<run_id>] Wrote requirements artifact: <path> (sha256: <8-char-prefix>...)`
- `gate1_waiting_approval`: `⏸️ [<run_id>] Waiting for Gate 1 approval`
- `research_agent_started`: `▶️ [<run_id>] Starting research agent`
- `research_finalized`: `✅ [<run_id>] Research finalized`
- `research_artifact_written`: `📝 [<run_id>] Wrote research artifact: <path> (sha256: <8-char-prefix>...)`
- `waiting_service_option_confirmations`: `⏸️ [<run_id>] Waiting for service option confirmations`
- `service_confirmations_artifact_written`: `📝 [<run_id>] Wrote service confirmations artifact: <path> (sha256: <8-char-prefix>...)`
- `gate2_waiting_approval`: `⏸️ [<run_id>] Waiting for Gate 2 approval`
- `gate1_marker_created`: `🔐 [<run_id>] Gate 1 marker created: .gate1.approved`
- `gate2_marker_created`: `🔐 [<run_id>] Gate 2 marker created: .gate2.approved`
- `authoring_agent_started`: `▶️ [<run_id>] Starting authoring agent`
- `authoring_finalized`: `✅ [<run_id>] Authoring finalized`
- `authoring_artifact_written`: `📝 [<run_id>] Wrote authoring artifact: <path> (sha256: <8-char-prefix>...)`
- `validator_agent_started`: `▶️ [<run_id>] Starting validator agent`
- `validator_pass`: `✅ [<run_id>] Validator PASS`
- `validator_fail`: `❌ [<run_id>] Validator FAIL (cycle <n>/3)`
- `remediation_cycle_started`: `🔁 [<run_id>] Remediation cycle <n>/3 started`
- `blocking_issues_remain`: `🧱 [<run_id>] Blocking issues remain: <top 3>`
- `validator_marker_created`: `🏁 [<run_id>] Validator marker created: .validator.passed`
- `duplicate_completion_ignored`: `ℹ️ [<run_id>] Duplicate completion ignored for <phase>`
- `requirements_summary_rendered`: `🧾 [<run_id>] Requirements summary rendered`
- `research_summary_rendered`: `🧾 [<run_id>] Research summary rendered`
- `authoring_summary_rendered`: `🧾 [<run_id>] Authoring summary rendered`
- `validator_summary_rendered`: `🧾 [<run_id>] Validator summary rendered`

### Compact Markdown Templates

These templates are normative. Root MUST render the exact section structure for each phase summary.
Any deviation in required sections/fields is a contract violation.

`requirements` rendering pattern:

```markdown
## 📋 Requirements Summary

- run_id: `<run_id>`
- artifact: `<artifact_path>`

### 🧭 Overview
<requirements_summary>

### ⚙️ Profile
- location/env/project: <azure_location> / <environment> / <project_name>
- demand/availability/cost: <workload_demand_profile> / <availability_target> / <cost_objective>

### ✅ Requirements
- functional (<count>): <top items...>
- non-functional (<count>): <top items...>
- constraints: <top items...>

### ⚠️ Risk And Status
- risk_notes: <top items...>
- open_questions: `None` or `<count> unresolved`
```

`research` rendering pattern:

```markdown
## 🔎 Research Summary

- run_id: `<run_id>`
- artifact: `<artifact_path>`

### 🧾 Evidence
<evidence_summary>

### 📐 Coverage
- requirement_mapping count: <count>
- citations count: <count>

### ⚠️ Gaps And Implications
- conflicts_or_gaps: <top items...>
- implementation_implications: <top items...>
```

`code` rendering pattern (`authoring` event/artifact contract IDs):

```markdown
## 🛠️ Authoring Summary

- run_id: `<run_id>`
- artifact: `<artifact_path>`

### 📦 Output
- files_written (<count>): <key paths...>
- avm_modules: <top items...>
- selected_service_options: <top items...>

### 🚦 Status
- blockers: `None` or `<top blockers...>`
```

`validator` rendering pattern:

```markdown
## 🧪 Validator Summary

- run_id: `<run_id>`
- artifact: `<artifact_path>`

### 🧾 Result
- validation_status: `PASS` or `FAIL`
- check_summary: <text>
- validated_files count: <count>

### 🚨 Findings
- failed_checks: `None` or <top failed checks...>
- blockers: `None` or <top blockers...>
```

- `requirements` summary must include:
  - `requirements_summary`
- `deployment_profile`
  - counts and top items for `functional_requirements` and `non_functional_requirements`
  - `constraints`
  - `risk_notes`
  - `open_questions` status
- `research` summary must include:
  - `evidence_summary`
  - `requirement_mapping` count
  - top `conflicts_or_gaps`
  - top `implementation_implications`
  - `citations` count
- `code` summary must include:
  - `files_written` count and key paths
  - `avm_modules`
  - `selected_service_options`
  - `blockers` status
- `validator` summary must include:
  - `validation_status` (`PASS` or `FAIL`)
  - top failed checks when present
  - `check_summary`
  - `blockers`
  - `validated_files` count

## Coding Agent Handoff Contract

When spawning `code`, send an `INIT_CODE` envelope payload that includes:

- `requirements_artifact_path`
- `research_evidence_artifact_path`
- `service_confirmations_artifact_path`
- `approval_markers` with marker paths (`gate1_marker_path`, `gate2_marker_path`)
- `destination_path` (always `output/<run_id>/infra/`)
- optional `mode` (`full` default; `preflight` or `remediation` only when explicitly needed)
- optional remediation context (`validator_evidence_artifact_path`, `remediation_blockers`)

Do not pass raw conversation transcript. Pass only normalized artifact paths and implementation intent.
Treat `service.confirmations.v2.json` as the sole source of truth for user-confirmed service choices.

## Validator Agent Handoff Contract

When spawning `validate`, send an `INIT_VALIDATE` envelope payload that includes:

- `requirements_artifact_path`
- `research_evidence_artifact_path`
- `service_confirmations_artifact_path`
- `authoring_evidence_artifact_path`
- `approval_markers` with marker paths (`gate1_marker_path`, `gate2_marker_path`)
- `paths_to_validate` (one or more authored destination paths)
- optional `validation_mode` (`strict` default)

Do not pass raw conversation transcript. Pass only normalized artifact paths and validation intent.
Treat `service.confirmations.v2.json` as the sole source of truth for user-confirmed service choices.

## Mandatory Trigger And Fail-Closed Enforcement

- Trigger this workflow for any Azure infrastructure intent, including create/update/delete/deploy/configure operations.
- Do not execute Azure-mutating operations (MCP Azure writes, `az` create/update/delete, IaC deploy) until:
  - Step 6 is complete, and
  - Approval Gate 1 is explicitly approved by user, and
  - Step 11 is complete, and
  - Step 12 service-choice confirmations are complete, and
  - Approval Gate 2 is explicitly approved by user, and
  - Step 17 is complete, and
  - Step 22 is complete.
- Do not execute Bicep code mutations until:
  - Step 6 is complete, and
  - Step 11 is complete, and
  - Step 12 service-choice confirmations are complete, and
  - Approval Gate 2 is explicitly approved by user.
- If validator status is `FAIL`, stop workflow and require remediation loop (max 3 cycles) before any Azure mutation.
- If a request arrives and these preconditions are not met, do not proceed with direct Azure actions. Start at Runtime Flow step 1.
- Record gate approvals under `output/<run_id>/`:
  - `.gate1.approved`
  - `.gate2.approved`
- Treat missing approval marker files as a hard stop for Azure mutation.

## Preflight Checklist (Must Pass Before Azure Mutation)

- `run_id` exists and matches `YYYYMMDD_project-slug_env_xxxx`.
- `requirements.v2.json` exists and validates against `.codex/protocols/artifacts/requirements-artifact.v2.schema.json`.
- `research.evidence.v2.json` exists and validates against `.codex/protocols/artifacts/research-evidence-artifact.v2.schema.json`.
- `service.confirmations.v2.json` exists and validates against `.codex/protocols/artifacts/service-confirmations-artifact.v2.schema.json`.
- `service.confirmations.v2.json` contains the explicit user-selected service options or an empty validated selection set when no confirmations were required.
- `authoring.evidence.v2.json` exists and validates against `.codex/protocols/artifacts/authoring-evidence-artifact.v2.schema.json`.
- `validator.evidence.v2.json` exists and validates against `.codex/protocols/artifacts/validator-evidence-artifact.v2.schema.json`.
- `progress.events.v2.json` exists and validates against `.codex/protocols/artifacts/progress-events-artifact.v2.schema.json`.
- `.validator.passed` exists.
- Approval marker files `.gate1.approved` and `.gate2.approved` exist.
- Authoring destination path resolves to `output/<run_id>/infra/`.

## Error Handling

- For invalid envelope/payload:
  - Return `ERROR` to user with validation reason.
  - Allow one retry request to subagent when recoverable.
- If retry fails, stop and surface failure with artifact paths and next-step guidance.
- For validator `FAIL`:
  - Run remediation loop with `code` then `validate` recheck.
  - Maximum remediation cycles: 3.
  - If still failing after 3 cycles, stop and surface blockers with artifact paths.

## Guardrails

- Never pass raw conversation transcripts to subagents.
- Never send non-envelope text to subagents.
- Never allow subagents to ask plain-text interactive questions.
- Never read `.codex/protocols/tests/**` from root orchestrator or any workflow subagent.
- Never emit ad-hoc root progress lines; use only cataloged progress `event_id` templates.
- Never bypass approval gates.
- Never bypass service-choice confirmation when research provides multiple viable service options.
- Never allow `code` to ask user questions, perform research, or override confirmed service selections.
- Never allow `validate` to author infrastructure code, perform research lookups, or downgrade strict failures to warnings.
- Never infer confirmed service selections from research prose when `service.confirmations.v2.json` is available.
- Never wait for subagent completion without using `wait` and explicit `timeout_ms`.
- Never allow any phase wait window to exceed 15 minutes.
- Never leave a completed phase agent open; close with `close_agent` after first valid completion.
- Never use `send_input` when agent state is paused/waiting.
- Never use `resume_agent` when agent state is active.
- Never continue when agent thread state is unknown or mismatched.
- Never request Gate 1/2 approval, start the next phase agent, or finalize workflow status before emitting the required phase summary-rendered event.

## Phase/File Allowlist

- Default mode is deny-by-default for file reads and tool usage outside explicit allowlists.
- Root MUST use `.codex/protocols/contract-min.v2.json` as the primary contract source during runtime orchestration.
- Root MUST NOT load full schemas unless one of these conditions is true:
  - envelope/artifact validation requires fields not present in `contract-min.v2.json`
  - contract/runtime failure debugging is explicitly active
- Unless explicitly marked `Subagent-only reads`, allowlist entries apply to root orchestrator.
- Any read/tool action not explicitly allowlisted is a contract violation and must fail closed with `ERROR`.
- Root collab tool allowlist across all phases:
  - `spawn_agent`
  - `wait`
  - `close_agent`
  - `send_input`
  - `resume_agent`
  - `request_user_input`

### Requirements Phase (Steps 1-7)

- Allowed reads:
  - `.codex/protocols/contract-min.v2.json`
  - `.codex/protocols/workflow-envelope.v2.schema.json`
  - `.codex/protocols/requirements.v2.schema.json`
  - `.codex/protocols/artifacts/requirements-artifact.v2.schema.json`
  - `.codex/agents/instructions/root_orchestrator.md`
  - `.codex/agents/instructions/requirements.md`
  - `output/<run_id>/**`
- Forbidden reads:
  - `.codex/protocols/tests/**`
  - Repo-wide recursive discovery reads (`Get-ChildItem -Recurse`, `rg --files`).
- Tool restrictions:
  - Root orchestrator and requirements subagent must not call MCP tools in this phase.
  - Root orchestrator may use only the root collab tool allowlist for orchestration and user confirmation.

### Research Phase (Steps 8-13)

- Allowed reads:
  - `.codex/protocols/contract-min.v2.json`
  - `.codex/protocols/workflow-envelope.v2.schema.json`
  - `.codex/protocols/research.v2.schema.json`
  - `.codex/protocols/artifacts/research-evidence-artifact.v2.schema.json`
  - `.codex/protocols/artifacts/service-confirmations-artifact.v2.schema.json`
  - `.codex/agents/instructions/root_orchestrator.md`
  - `.codex/agents/instructions/research.md`
  - `output/<run_id>/**`
- Forbidden reads:
  - `.codex/protocols/tests/**` (no exceptions, including debug mode)
- Tool restrictions:
  - Only `research` may use Microsoft Learn MCP.
  - Root orchestrator must not call Azure/Microsoft Learn/Bicep/OpenAI MCP tools directly.
  - Root orchestrator may use only the root collab tool allowlist for orchestration and user confirmation.

### Code Phase (Steps 14-17)

- Allowed reads:
  - `.codex/protocols/contract-min.v2.json`
  - `.codex/protocols/code.v2.schema.json`
  - `.codex/protocols/artifacts/authoring-evidence-artifact.v2.schema.json`
  - `.codex/protocols/artifacts/service-confirmations-artifact.v2.schema.json`
  - `.codex/agents/instructions/root_orchestrator.md`
  - `.codex/agents/instructions/code.md`
  - `output/<run_id>/requirements.v2.json`
  - `output/<run_id>/research.evidence.v2.json`
  - `output/<run_id>/service.confirmations.v2.json`
  - `output/<run_id>/.gate1.approved`
  - `output/<run_id>/.gate2.approved`
- Subagent-only reads:
  - `.codex/skills/bicep-avm-code/**`
- Forbidden reads:
  - `.codex/protocols/tests/**`
- Tool restrictions:
  - Only `code` performs code implementation in this phase.
  - `code` may use Bicep MCP tools, but must not use Microsoft Learn tools or interactive user-input flows.
  - Root orchestrator must not load `.codex/skills/**` in this phase.
  - Root orchestrator may use only the root collab tool allowlist for orchestration and user confirmation.

### Validation Phase (Steps 18-22)

- Allowed reads:
  - `.codex/protocols/contract-min.v2.json`
  - `.codex/protocols/validate.v2.schema.json`
  - `.codex/protocols/artifacts/validator-evidence-artifact.v2.schema.json`
  - `.codex/protocols/artifacts/service-confirmations-artifact.v2.schema.json`
  - `.codex/agents/instructions/root_orchestrator.md`
  - `.codex/agents/instructions/validate.md`
  - `output/<run_id>/requirements.v2.json`
  - `output/<run_id>/research.evidence.v2.json`
  - `output/<run_id>/service.confirmations.v2.json`
  - `output/<run_id>/authoring.evidence.v2.json`
  - `output/<run_id>/.gate1.approved`
  - `output/<run_id>/.gate2.approved`
- Subagent-only reads:
  - `.codex/skills/bicep-avm-validate/**`
- Forbidden reads:
  - `.codex/protocols/tests/**`
- Tool restrictions:
  - `validate` may use Bicep tools and local repository evidence only.
  - `validate` must not call Microsoft Learn tools or Azure search tools.
  - `validate` must not write infrastructure files or marker artifacts; it may only write validator evidence artifacts.
  - Root orchestrator must not load `.codex/skills/**` in this phase.
  - Root orchestrator may use only the root collab tool allowlist for orchestration and user confirmation.
