---
name: bicep-avm-code
description: Author, update, refactor, and remediate Azure Bicep using Azure Verified Modules (AVM). Use when Codex needs to implement infrastructure from a design brief, structured handoff artifact, or explicit task context, convert existing Bicep to AVM-first patterns, repair dependency or layout issues, or fix Bicep authoring blockers. Do not use for research-only tasks or validation-only review.
---

# Bicep AVM Code

Implement Azure infrastructure in Bicep with an AVM-first posture. Keep this skill focused on authoring and remediation. Do not re-run architecture research, and do not act as the final validation gate.

## Source of truth order

Use this order whenever inputs disagree:

1. design brief, structured handoff artifact, or explicit task context
2. `mcp__bicep__list_avm_metadata`
3. `mcp__bicep__get_bicep_best_practices`
4. `mcp__bicep__get_bicep_file_diagnostics`, `bicep build`, and `bicep lint`
5. The reference files in this skill

## Prerequisite check

Before authoring:

```powershell
bicep --version
```

Required tools:

- `mcp__bicep__list_avm_metadata`
- `mcp__bicep__get_bicep_best_practices`
- `mcp__bicep__get_bicep_file_diagnostics`
- `bicep build <entrypoint.bicep>`
- `bicep lint <entrypoint.bicep>`

## Quick start

Author in this order:

1. Resolve AVM coverage and versions with one metadata snapshot.
2. Consult `references/producer-owned-capabilities.md` for the selected producer modules.
3. Author or update the entrypoint with direct AVM modules and producer-owned capabilities.
4. Apply security, naming, networking, and dependency guardrails.
5. Run diagnostics, build, lint, and the validation contract checks.
6. If authored-code blockers remain, remediate them and re-run the validation checks.
7. Stop after at most 2 remediation loops; if blockers still remain after the second loop, report them as unresolved.

## Metadata and documentation discipline

- Run `mcp__bicep__list_avm_metadata` once at the start of module resolution for the task.
- Treat that first response as the working registry snapshot for the whole turn; cache the selected module paths, versions, and `documentationUri` values in your working notes and reuse them.
- Do not re-run `mcp__bicep__list_avm_metadata` unless one of these exceptions applies:
  - `new_service_scope`: a newly introduced Azure service was not part of the original required resource set.
  - `unresolved_registry_gap`: the initial metadata response did not contain the needed module entry.
- When a selected metadata entry includes `documentationUri`, use that URI directly for module parameter and capability lookup.
- Do not run a fresh README, module-name, or version web search when `documentationUri` is already present.
- Use external documentation lookup only when one of these exceptions applies:
  - `missing_doc_uri`: the metadata entry does not include `documentationUri`.
  - `broken_doc_uri`: the `documentationUri` cannot be opened or is clearly stale or incomplete for the pinned version.

## Quick decision tree

```text
Need to author from a structured handoff artifact?
-> Read references/handoff-inputs.md

Need to convert existing Bicep to AVM?
-> Read references/avm-resolution.md
-> Read references/producer-owned-capabilities.md

Need to add or update Azure service wiring?
-> Read references/avm-resolution.md
-> Read references/producer-owned-capabilities.md
-> Then read references/security-naming-networking.md

Need to fix validation blockers or dependency cycles?
-> Read references/remediation.md
-> Use the `bicep-avm-validate` skill as the validation gate

Need to handle a missing AVM?
-> Read references/avm-resolution.md
```

## Reference map

Open only what you need:

- `references/handoff-inputs.md` -> authoring from a brief or structured handoff artifact, input precedence, remediation cycles
- `references/authoring-contract.md` -> entrypoint contract, parameter rules, anti-patterns, completion bar
- `references/avm-resolution.md` -> AVM lookup order, targeted upstream capability proof, and exception rules
- `references/producer-owned-capabilities.md` -> normative catalog of producer-owned AVM parameters that must replace sibling resources or helper modules
- `references/security-naming-networking.md` -> shared security, naming, tagging, subnet, and exposure guardrails
- `references/remediation.md` -> validator blocker handling, dependency-cycle repair, and fail-closed exception re-checks
- `../bicep-avm-validate/SKILL.md` -> implementation contract for the `bicep-avm-validate` skill; use that skill as the internal validation gate after authoring and after each remediation pass, not as a separate review handoff

## Validation and remediation loop

- Default to an internal `author -> validate -> remediate -> re-validate` loop before handoff.
- Use the `bicep-avm-validate` skill for each validation pass, following the contract in `../bicep-avm-validate/SKILL.md`.
- Count only remediation passes toward the cap; the initial validation after authoring does not consume a loop.
- Stop after 2 remediation loops maximum.
- If blockers remain after loop 2, do not start a third remediation pass; report the remaining blockers and the attempted fixes.
- When the user explicitly asks for validation-only review, do not use this loop; use `bicep-avm-validate` instead.

## Guardrails

- Use direct `br/public:avm/...` references for AVM-covered resources. Treat native resources, local wrappers, or sibling AVM resource modules for those services as blockers unless an exception is recorded.
- Treat repeated AVM metadata lookups and ad-hoc README or web searches as blockers unless a named metadata or documentation exception is recorded.
- Keep fixed design choices in module properties and logic, not decision-proxy parameters.
- Use parameters only for runtime, secret, or operational inputs.
- Prefer managed identity over secrets, keys, and connection strings.
- Use `references/producer-owned-capabilities.md` as the normative source for producer-owned child and adjunct capabilities.
- Treat sibling resources or standalone AVM helper/PTN modules for cataloged producer-owned capabilities as blockers unless the catalog documents a cycle or scope exception.
- Avoid `consumer -> producer -> consumer` cycles; derive names and IDs when possible.
- Run capability proof only for a specific producer module plus a specific capability inferred from declared fallback code or a validator blocker.
- For uncataloged modules, allow generic direct-AVM authoring of the producer resource but do not invent local capability rules or fallback sibling resources from memory.
- Use native resources or standalone helper/PTN modules only when targeted proof shows the producer AVM lacks the capability or a real cycle/scope exception is recorded.
- Never use restored caches, `main.json`, recursive source extraction, or broad param hunting as capability proof.
- Do not skip diagnostics, build, or lint.
- Do not skip the internal validation gate after authoring or after a remediation pass.
- When the brief’s intent language signals “private,” “corp,” “no public access,” or similar, assume every AVM-capable resource needs producer-owned private endpoints (and Private DNS if available) and ensure `publicNetworkAccess` is disabled unless a documented exception permits mixed access.
- Never clone or download repositories or files locally just to research AVM modules or documentation; rely only on sanctioned tools and references.

## Exclusions

Do not use this skill to:

- perform architecture research or service selection
- ask the user to choose SKUs that should already be resolved
- act as the final validation sign-off
- replace explicit task context with inferred product choices
