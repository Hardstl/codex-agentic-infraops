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
- This workflow covers root-owned requirements intake, Microsoft Learn research, post-approval AVM Bicep code implementation, and post-code validation.

## Canonical Contract Source

All subagent messages must validate against these schemas:

- `.codex/protocols/workflow-envelope.v2.schema.json`
- `.codex/protocols/research.v2.schema.json`
- `.codex/protocols/code.v2.schema.json`
- `.codex/protocols/validate.v2.schema.json`

Root-owned requirements payloads must validate against:

- `.codex/protocols/requirements.v2.schema.json`

All persisted artifacts must validate against:

- `.codex/protocols/artifacts/requirements-artifact.v2.schema.json`
- `.codex/protocols/artifacts/research-evidence-artifact.v2.schema.json`
- `.codex/protocols/artifacts/service-confirmations-artifact.v2.schema.json`
- `.codex/protocols/artifacts/authoring-evidence-artifact.v2.schema.json`
- `.codex/protocols/artifacts/validator-evidence-artifact.v2.schema.json`
- `.codex/protocols/artifacts/progress-events-artifact.v2.schema.json`

Prompt text is not a contract source of truth.

## Root Responsibilities

- Collect structured user input:
  - Use `request_user_input` when available with one decision per question and stable `question.id` keys.
  - Preserve a plain-text fallback path and map fallback responses to the same question ids.
  - Map friendly user-facing option labels to canonical values using a fixed lookup keyed by (`question.id`, selected `label`).
  - When `Other` is chosen for enum-backed question ids, collect free text and map by case-insensitive alias to canonical enums; if no canonical match, re-prompt with valid choices.
- Own requirements intake directly in root:
  - Collect exactly 2 rounds of inputs with stable ids: Round 1 uses `azure_location`, `project_name`, `environment`; Round 2 uses `workload_demand_profile`, `cost_objective`, `availability_target`.
  - Preserve the existing Round 2 question texts and option labels so the intake remains deterministic across runs.
  - Normalize answers to canonical values and synthesize the requirements payload directly in root; do not spawn a `requirements` subagent.
  - Keep this step intake-scoped only; do not perform service selection, implementation discovery, or Microsoft Learn analysis while synthesizing requirements.
- Manage approval gates with structured root prompts:
  - Use `request_user_input` with one decision per question and stable `question.id` keys.
  - Keep the recommended option first for every question that has options.
  - Preserve a plain-text fallback path and map fallback responses to the same question ids.
- Enforce per-service option confirmation before any code/deployment planning that depends on service SKU/tier choices.
- Spawn and coordinate subagents with an explicit lifecycle:
  - Use `spawn_agent` for each active phase agent (`research`, `code`, `validate`) and persist the returned `thread_id`.
  - Use `wait` as the only phase-blocking primitive with `timeout_ms=30000`.
  - On first valid phase completion, mark phase complete and call `close_agent` for that `thread_id`.
  - For validator-driven remediation, always spawn a fresh `code` agent in `mode=remediation`; never reuse a completed `code` thread.
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
- Attempt to persist semantic root progress events to `output/<run_id>/progress.events.v2.json` on a best-effort basis.
- For compact terminal envelopes, validate `artifact_path` and `artifact_sha256` before accepting completion.
- Render short best-effort terminal summaries in root CLI for finalized artifacts (`requirements`, `research`, `code`, `validator`) after validation/persist.
- Deduplicate repeated completion notifications and never echo full payloads when a compact summary is already available.
- Convert structured research `service_confirmation_requests` into `request_user_input` questions and persist validated selections to `output/<run_id>/service.confirmations.v2.json`.
- Treat `selection` and `forced_confirmation` service-confirmation requests differently:
  - For `selection`, map the provided service options into one `request_user_input` decision with the recommended option first.
  - For `forced_confirmation`, map the sole compliant option into a confirm-or-stop decision; if the user declines, fail closed and stop authoring progression until constraints are revisited.
- Spawn `code` only after Gate 2 approval and only with finalized artifact inputs.
- Spawn `validate` after code implementation and enforce fail-closed validation loop.
- Stop on unrecoverable contract violations.

## Root Read/Tool Minimalism

- Root orchestrator is control-plane only: orchestration, gating, envelope/schema checks, artifact persistence, and CLI summaries.
- Root must not perform implementation discovery, AVM capability analysis, or validation-rule interpretation on behalf of subagents.
- Root must not read `.codex/skills/**` during normal workflow execution.
- Root may read `.codex/skills/**` only when explicitly debugging a contract/runtime failure and must return to minimal reads afterward.
- Root MUST prefer compact envelope previews/counts and MUST NOT load full artifacts unless required for schema/checksum verification or unrecoverable error handling.
- Root MUST NOT inline duplicate normalized requirements content into `INIT_RESEARCH`; pass the validated requirements artifact path instead.

## Runtime Flow

1. Collect requirements Round 1 inputs in root CLI using `request_user_input`:
   - `azure_location`
   - `project_name`
   - `environment`
2. Create `run_id` (`YYYYMMDD_project-slug_env_xxxx`) using the selected `project_name` and `environment`.
3. Create `output/<run_id>/`.
   - Initialize `output/<run_id>/progress.events.v2.json` with a best-effort semantic-event scaffold when progress persistence is available.
4. Collect requirements Round 2 inputs in root CLI using `request_user_input`:
   - `workload_demand_profile`
   - `cost_objective`
   - `availability_target`
   - Use these exact question texts and labels:
     - `workload_demand_profile`: `How much traffic/load do you expect in the next 3 months?`
     - labels: `Low demand`, `Medium demand`, `High demand`
     - `cost_objective`: `Which cost strategy should we optimize for?`
     - labels: `Cost-first (single-zone risk accepted)`, `Balanced (cost and resilience)`, `Performance-first (higher spend acceptable)`
     - `availability_target`: `What availability target do you need for this workload?`
     - labels: `Cost-first (single-zone risk accepted)`, `Balanced (zone-resilient default)`, `Resilience-first (regional failover target)`
5. Normalize the 6 intake answers into a requirements payload, validate it against `.codex/protocols/requirements.v2.schema.json`, then validate and write `output/<run_id>/requirements.v2.json` against `.codex/protocols/artifacts/requirements-artifact.v2.schema.json`.
   - Attempt a short best-effort terminal summary using validated artifact data; continue even if rendering fails.
6. Approval Gate 1:
   - Request explicit user confirmation with `request_user_input` (single question, one decision, stable id, recommended option first).
   - If `request_user_input` is unavailable, ask plain-text confirmation and map to the same id.
7. Call `spawn_agent` for `research` with `fork_context=false`.
   - Persist run-state for the research thread using the same required fields.
8. Send `INIT_RESEARCH` envelope with `requirements_artifact_path` only.
9. Wait for `FINAL_RESEARCH_EVIDENCE`.
   - Use `wait` with `timeout_ms=30000` and enforce the same 15-minute max elapsed wait.
   - On first valid completion, accept once and call `close_agent` for the research `thread_id`.
   - Ignore duplicate completion notifications after first acceptance.
10. Validate compact research envelope fields (`artifact_path`, `artifact_sha256`, preview/count bounds, optional `service_confirmation_requests` bounds), then validate artifact schema and checksum for `output/<run_id>/research.evidence.v2.json`.
   - Attempt a short best-effort terminal summary using validated artifact data; continue even if rendering fails.
11. Inspect research `service_confirmation_requests`.
   - If one or more requests are present, map each `selection` request into a `request_user_input` question with the provided options and recommended choice first.
   - Map each `forced_confirmation` request into a `request_user_input` confirm-or-stop question:
     - `Confirm recommended option (Recommended)` confirms the sole compliant option and records its id/label in `service.confirmations.v2.json`.
     - `Stop for revision` fails closed and stops workflow progression until the user revisits the relevant constraints or requirements.
   - Keep one decision per question and stable question ids; if `request_user_input` is unavailable, collect plain-text fallback responses mapped to the same ids.
   - Collect explicit selections in root CLI only; never ask the user for combined free-text codes such as `1A, 2B, 3C`.
   - Validate and write `output/<run_id>/service.confirmations.v2.json` even when the request list is empty.
12. Approval Gate 2:
   - Request explicit user confirmation with `request_user_input` (single question, one decision, stable id, recommended option first).
   - If `request_user_input` is unavailable, ask plain-text confirmation and map to the same id.
13. Call `spawn_agent` for `code` with `fork_context=false`.
   - Persist run-state for the code thread using the same required fields.
14. Send `INIT_CODE` envelope with normalized artifact paths, gate marker paths, and destination path.
15. Wait for `FINAL_AUTHORING_EVIDENCE`.
   - Use `wait` with `timeout_ms=30000` and enforce the same 15-minute max elapsed wait.
   - On first valid completion, accept once and call `close_agent` for the code `thread_id`.
   - Ignore duplicate completion notifications after first acceptance.
16. Validate compact code/authoring envelope fields (`artifact_path`, `artifact_sha256`, preview/count bounds), then validate artifact schema and checksum for `output/<run_id>/authoring.evidence.v2.json`.
   - Attempt a short best-effort terminal summary using validated artifact data; continue even if rendering fails.
17. Call `spawn_agent` for `validate` with `fork_context=false`.
   - Persist run-state for the validate thread using the same required fields.
18. Send `INIT_VALIDATE` envelope with normalized artifact paths, gate marker paths, and authored destination path(s).
19. Wait for `FINAL_VALIDATOR_EVIDENCE`.
   - Use `wait` with `timeout_ms=30000` and enforce the same 15-minute max elapsed wait.
   - On first valid completion, accept once and call `close_agent` for the validate `thread_id`.
   - Ignore duplicate completion notifications after first acceptance.
20. Validate compact validator envelope fields (`artifact_path`, `artifact_sha256`, `validation_status`, preview/count bounds), then validate artifact schema and checksum for `output/<run_id>/validator.evidence.v2.json`.
   - Attempt a short best-effort terminal summary using validated artifact data; continue even if rendering fails.
21. If validator fails, run remediation loop (max 3 cycles):
   - Emit `remediation_cycle_started` for the current cycle.
   - Call `spawn_agent` for a fresh `code` thread with `fork_context=false` and persist new code run-state fields for that cycle.
   - Send `INIT_CODE` with the same normalized artifact paths, gate marker paths, and destination path, plus `mode=remediation`, `validator_evidence_artifact_path`, and `remediation_blockers`.
   - Wait for `FINAL_AUTHORING_EVIDENCE`, then validate, persist, summarize, and close that remediation code thread using the same lifecycle and timeout rules as Steps 15-16.
   - Call `spawn_agent` for a fresh `validate` thread with `fork_context=false`.
   - Send `INIT_VALIDATE` with normalized artifact paths, gate marker paths, and authored destination path(s).
   - Wait for `FINAL_VALIDATOR_EVIDENCE`, then validate, persist, summarize, and close that validator thread using the same lifecycle and timeout rules as Steps 19-20.
   - If validator status remains `FAIL` after a cycle and fewer than 3 cycles have run, continue to the next remediation cycle.
   - If validator status remains `FAIL` after cycle 3, stop and surface blockers with artifact paths.
   - On validator pass, root writes `output/<run_id>/.validator.passed`.

## CLI Presentation Contract

- Terminal summaries and progress lines are presentation-only and non-contractual.
- JSON artifacts, checksums, approvals, and schema-validated envelopes remain the sole source of truth for workflow execution.
- Do not persist Markdown artifact files; print summaries in root CLI only.
- Render terminal output on a best-effort basis:
  - prefer compact, readable text with short headings or icons when helpful
  - include `run_id` and artifact path when summarizing finalized artifacts
  - keep summaries short and derived only from already-validated data
- If terminal rendering fails, emit a warning when feasible and continue workflow execution.
- Never pass rendered terminal text to subagents; pass only schema-valid envelopes and normalized artifact paths.
- On repeated completion events, keep only the first valid completion per phase and ignore duplicates.
- When rendering service confirmations in root, prefer one question per service decision and place the recommended option first.
- For `forced_confirmation`, render a confirm-or-stop question rather than inventing placeholder service alternatives.

### Progress Events (Root CLI)

- Emit semantic progress events for major phase transitions and artifact-write milestones.
- Progress events should use cataloged `event_id` values from `.codex/protocols/progress-event-catalog.v2.json`.
- Persist progress events to `output/<run_id>/progress.events.v2.json` when possible, but do not block workflow execution if rendering or persistence fails.
- Progress entries should remain compact and structured:
  - persist `event_id`, `timestamp_utc`, and minimal placeholders/metadata needed to interpret the event
  - do not persist full envelope payloads or long prose summaries
- Derive short human-readable terminal text from semantic events on a best-effort basis.
- If an `event_id` is unknown or a progress entry cannot be persisted cleanly, warn when feasible and continue workflow execution.

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
For remediation, root MUST spawn a new `code` agent and set `mode=remediation`; completed code threads are never resumed.

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
  - Step 5 is complete, and
  - Approval Gate 1 is explicitly approved by user, and
  - Step 10 is complete, and
  - Step 11 service-choice confirmations are complete, and
  - Approval Gate 2 is explicitly approved by user, and
  - Step 16 is complete, and
  - Step 21 is complete.
- Do not execute Bicep code mutations until:
  - Step 5 is complete, and
  - Step 10 is complete, and
  - Step 11 service-choice confirmations are complete, and
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
- `.validator.passed` exists.
- Approval marker files `.gate1.approved` and `.gate2.approved` exist.
- Authoring destination path resolves to `output/<run_id>/infra/`.

## Error Handling

- For invalid envelope/payload:
  - Return `ERROR` to user with validation reason.
  - Allow one retry request to subagent when recoverable.
- If retry fails, stop and surface failure with artifact paths and next-step guidance.
- For validator `FAIL`:
  - Run remediation loop with a fresh `code` spawn in `mode=remediation`, then a fresh `validate` recheck.
  - Maximum remediation cycles: 3.
  - If still failing after 3 cycles, stop and surface blockers with artifact paths.

## Guardrails

- Never pass raw conversation transcripts to subagents.
- Never send non-envelope text to subagents.
- Never allow subagents to ask plain-text interactive questions.
- Never read `.codex/protocols/tests/**` from root orchestrator or any workflow subagent.
- Never let terminal rendering or progress persistence control phase advancement, approvals, or final workflow status.
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
- Never send remediation blockers to a completed `code` thread; spawn a new remediation `code` agent instead.
- Never continue when agent thread state is unknown or mismatched.

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

### Requirements Intake Phase (Steps 1-6)

- Allowed reads:
  - `.codex/protocols/contract-min.v2.json`
  - `.codex/protocols/requirements.v2.schema.json`
  - `.codex/protocols/artifacts/requirements-artifact.v2.schema.json`
  - `.codex/agents/instructions/root_orchestrator.md`
  - `output/<run_id>/**`
- Forbidden reads:
  - `.codex/protocols/tests/**`
  - Repo-wide recursive discovery reads (`Get-ChildItem -Recurse`, `rg --files`).
- Tool restrictions:
  - Root orchestrator must not call MCP tools in this phase.
  - Root orchestrator may use only the root collab tool allowlist for orchestration, user confirmation, and deterministic requirements intake.

### Research Phase (Steps 7-12)

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

### Code Phase (Steps 13-16)

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

### Validation Phase (Steps 17-21)

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
