# Agent Setup Review Tracker

This document is the working tracker for the agent-setup review. It captures the current findings, the task order, and the one-task-at-a-time workflow we will use to make changes and mark progress.

## Current Findings Summary

- Tasks 1-5 are now implemented.
- The workflow keeps the `requirements.v2.json` artifact boundary, but requirements intake is now owned directly by root instead of a dedicated phase agent.
- Research, code, and validation remain as the active subagent phases.

## Working Rules

- Move through the checklist in severity order unless a completed task reveals a dependency that changes the sequence.
- Only one task may be `in progress` at a time.
- Make only the changes needed for the active task.
- Validate the active task before marking it complete.
- Record follow-on notes in the task section before moving to the next item.

## Task Checklist

- [x] Task 1: Fix the remediation-loop lifecycle contradiction in the root orchestrator.
- [x] Task 2: Fix the research single-option confirmation contradiction between prompt and schema.
- [x] Task 3: Remove presentation/rendering requirements from the fail-closed execution path, or clearly demote them to non-blocking behavior.
- [x] Task 4: Slim the `INIT_RESEARCH` handoff so research gets either the artifact path or a minimal normalized summary, not both full forms.
- [x] Task 5: Reassess whether the requirements phase still justifies being its own agent once tasks 1-4 are complete.

## Task Details

### Task 1

Status: `completed`

Problem:
The root orchestrator closes phase agents on first valid completion, but the validator-fail path still assumes the code agent may remain active or resumable during remediation.

Target Change:
Make the remediation lifecycle internally consistent. Either keep the code agent alive across the validation boundary with explicit state rules, or always respawn it for remediation and remove the contradictory active/resume expectations.

Affected Contracts / Files:
- `.codex/agents/instructions/root_orchestrator.md`
- `.codex/protocols/code.v2.schema.json`
- `.codex/protocols/validate.v2.schema.json`

Validation Notes:
- Verify the root runtime flow uses one consistent lifecycle for initial authoring and remediation.
- Verify the recovery path can be followed without depending on an already-closed thread.

Follow-on Notes:
- Completed on 2026-04-14 after write access was restored.
- Root now standardizes remediation on fresh `code` and `validate` spawns instead of resuming a previously closed `code` thread.
- No schema edits were required for Task 1 because `.codex/protocols/code.v2.schema.json` already supports `mode=remediation`, `validator_evidence_artifact_path`, and `remediation_blockers`.

### Task 2

Status: `completed`

Problem:
The research prompt allows single-option confirmations for constrained cases, while the research schema requires at least two options.

Target Change:
Align the prompt and schema so constrained single-option confirmations are either explicitly supported everywhere or explicitly rejected everywhere.

Affected Contracts / Files:
- `.codex/agents/instructions/research.md`
- `.codex/protocols/research.v2.schema.json`

Validation Notes:
- Verify the schema and prompt agree on allowed confirmation counts.
- Verify the documented constrained-choice behavior is still fail-closed.

Follow-on Notes:
- Completed on 2026-04-14.
- We did not normalize the sole-option case into a single-item `options` array.
- Research contracts now distinguish between `selection` requests and `forced_confirmation` requests.
- Root is now expected to render `forced_confirmation` as a confirm-or-stop decision and persist the confirmed sole option into `service.confirmations.v2.json`.
- `service.confirmations.v2.json` did not need a schema change because it stores the final selected option, not the shape of the original research request.

### Task 3

Status: `completed`

Problem:
Markdown summary rendering and progress-event sequencing currently influence workflow success, even though they are presentation concerns.

Target Change:
Remove presentation duties from the blocking execution path, or narrow fail-closed behavior so it applies only to contractual data persistence and validation, not UI rendering concerns.

Affected Contracts / Files:
- `.codex/agents/instructions/root_orchestrator.md`
- `.codex/protocols/artifacts/progress-events-artifact.v2.schema.json`
- `.codex/protocols/progress-event-catalog.v2.json`

Validation Notes:
- Verify core workflow artifact persistence and schema validation remain mandatory.
- Verify rendering failures no longer invalidate otherwise-correct workflow state unless the contract still intentionally requires it.

Follow-on Notes:
- Completed on 2026-04-14.
- Root no longer gates phase progression on summary-render events or exact Markdown template output.
- Progress persistence remains available through `progress.events.v2.json`, but it is now semantic and best-effort rather than render-driven and fail-closed.
- The progress catalog is now a compact event registry with optional display hints instead of exact human-line templates.
- `.codex/protocols/contract-min.v2.json` was updated as part of Task 3 so the condensed contract matches the new non-blocking progress model.

### Task 4

Status: `completed`

Problem:
`INIT_RESEARCH` sends both `requirements_artifact_path` and the full normalized `requirements` object, creating avoidable duplication.

Target Change:
Reduce the research handoff to a single primary source plus only the minimum extra data needed for efficient reasoning.

Affected Contracts / Files:
- `.codex/protocols/research.v2.schema.json`
- `.codex/agents/instructions/root_orchestrator.md`
- `.codex/agents/instructions/research.md`

Validation Notes:
- Verify root and research agree on the new handoff shape.
- Verify research still has enough context to do targeted, decision-grade work without reintroducing duplication elsewhere.

Follow-on Notes:
- Completed on 2026-04-14.
- `INIT_RESEARCH` now uses `requirements_artifact_path` as the only required handoff field.
- Research now treats the validated requirements artifact as the sole source of truth and reads needed fields locally instead of receiving an inlined normalized duplicate.
- `.codex/protocols/contract-min.v2.json` was updated with the slimmer handoff so the condensed contract matches the new schema.

### Task 5

Status: `completed`

Problem:
The requirements phase may be providing a rigid questionnaire rather than enough synthesis value to justify an agent boundary.

Target Change:
Reassess the phase after tasks 1-4. Decide whether to keep it as-is, simplify it, or collapse it into root while preserving contract clarity and user-input handling.

Affected Contracts / Files:
- `.codex/agents/instructions/requirements.md`
- `.codex/protocols/requirements.v2.schema.json`
- `.codex/agents/instructions/root_orchestrator.md`

Validation Notes:
- Review the remaining context burden after tasks 1-4.
- Compare orchestration cost against actual reasoning value before proposing any architectural change.

Follow-on Notes:
- Completed on 2026-04-14.
- The dedicated requirements agent was collapsed into deterministic root-owned intake.
- The workflow still persists `output/<run_id>/requirements.v2.json`, so research/code/validate keep the same artifact handoff boundary.
- `.codex/protocols/requirements.v2.schema.json` now describes the normalized requirements payload rather than a subagent envelope exchange.
- `.codex/protocols/artifacts/requirements-artifact.v2.schema.json` now records `generated_by=root_orchestrator`.
- `.codex/config.toml` no longer registers a `requirements` agent for normal workflow execution.

## Decision Log / Assumptions

- The tracker lives at repo root as `AGENT_SETUP_REVIEW.md`.
- Tasks are ordered severity-first.
- This pass is intended to stabilize the current four-phase architecture before considering structural simplification.
- Task 5 is intentionally a reassessment task, not a pre-committed architecture change.
