# Handoff Inputs

Use this reference when authoring from a design brief, structured handoff artifact, or another agent's implementation package.

## Input precedence

- Read the handoff artifact or brief as the authoring contract.
- Use explicit resource decisions, networking expectations, identity requirements, and constraints first.
- Respect resources marked `existing` by wiring references instead of creating those resources.
- Treat stated blockers or unresolved decisions as hard stops. Do not fill gaps by re-running architecture selection inside this skill.

## Authoring outputs

- Write or update the requested Bicep entrypoint and supporting files.
- Record AVM modules used, files changed, validation results, and any remaining blockers in the task output expected by the surrounding workflow.

## Authoring loop

1. Read the brief or handoff artifact.
2. Resolve AVM coverage and versions for the selected services.
3. Author the env-specific entrypoint and supporting files.
4. Run diagnostics, build, and lint.
5. Return concrete authoring results in the format expected by the current workflow.

## Remediation loop

When a reviewer or validator returns blockers:

1. Fix only the stated blockers unless the fix requires a tightly related change.
2. Re-run diagnostics, build, and lint.
3. Refresh the task output with the new status.
4. Keep the contract stable unless the blocker exposed a contract mismatch.

## Boundaries

- Do not re-interpret an already normalized handoff artifact as if it were a fresh architecture brief.
- Do not perform architecture research or service selection inside this skill.
- Do not act as the final validation sign-off.
