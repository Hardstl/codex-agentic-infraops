# AVM Resolution

Use this reference when selecting AVM modules, updating versions, converting native Bicep to AVM, or handling missing AVM coverage.

## Resolution order

1. Read the required resource list from the task context, design brief, or structured handoff artifact.
2. Query `mcp__bicep__list_avm_metadata` once for the task, then filter that result to the required services.
3. Use direct `br/public:avm/...` references in the authored file for every AVM-covered resource.
4. Consult `references/producer-owned-capabilities.md` for the selected producer modules before authoring sibling resources or helper modules.
5. Run a targeted upstream capability check only when authored code shows a concrete producer-owned fallback candidate.
6. Use `mcp__bicep__get_bicep_best_practices` to catch patterns that the module choice might affect.

## Reuse-first rules

- Treat the first `mcp__bicep__list_avm_metadata` response as the working registry snapshot for the whole task.
- Reuse the selected module entries, pinned versions, and `documentationUri` values from that first response instead of re-querying.
- Treat repeat metadata lookup as an exception path, not normal flow.
- Allow repeat metadata lookup only for:
  - `new_service_scope`: a newly introduced Azure service was not part of the original discovery set.
  - `unresolved_registry_gap`: the initial metadata response did not contain the needed module entry.

## Documentation lookup rules

- Start module parameter and capability inspection from the selected metadata entry's `documentationUri`.
- When `documentationUri` exists, use it as the canonical first source for README and parameter lookup.
- Do not run GitHub or web searches for README pages, module names, or pinned versions when `documentationUri` is already present.
- Allow external documentation lookup only for:
  - `missing_doc_uri`: the metadata entry does not include `documentationUri`.
  - `broken_doc_uri`: the `documentationUri` cannot be opened or is clearly stale or incomplete for the pinned version.

## Versioning rules

- Pin explicit AVM versions in authored files.
- Prefer the latest stable version returned by the metadata lookup unless the task context or handoff artifact requires a pinned version.
- Keep a service family on a coherent set of versions when no task-specific reason exists to mix them.

## Targeted capability proof workflow

Before allowing a producer-owned capability exception:

1. Identify a concrete fallback candidate already declared in authored code.
2. Name the specific producer AVM and the specific capability in question.
3. Start from the pinned producer module's `documentationUri`; only if that is unavailable through `missing_doc_uri` or `broken_doc_uri`, fetch the pinned upstream `main.bicep` through a targeted follow-up.
4. Verify whether the required capability or parameter exists in that one file.

Rules:

- Do not run capability proof when no native or standalone adjunct fallback candidate exists.
- Do not scan modules just because they are present in the template or the brief mentions a feature.
- Use only the pinned producer module's canonical docs or top-level `main.bicep`.
- Never inspect restored caches, `source.tgz`, or cached `main.json`.
- If upstream fetch fails, treat the fallback as unverified.

## Conversion rules

- Replace native top-level resources with direct AVM module references when AVM coverage exists.
- Treat AVM-covered native resources as hard failures, not lower-quality alternatives.
- Collapse standalone extension resources and standalone AVM helper modules into producer module inputs when the capability catalog or targeted proof shows support.
- Keep exception paths only when the AVM lacks the required capability or a real cycle/scope constraint is recorded.
- If the producer module is cataloged, do not keep sibling resources or helper modules for listed capabilities.
- If the producer module is not cataloged, allow generic direct-AVM authoring of the producer resource but do not invent new local producer-owned capability rules or fallback sibling resources from memory.

## Fallback when AVM is missing

Use native Bicep only for the uncovered gap. When that happens:

1. Keep the rest of the solution AVM-first.
2. Use native Bicep only for the specific uncovered resource or capability after targeted proof shows the pinned AVM module lacks support.
3. Record the verification result and exception in the work output or task notes.
4. Avoid inventing local wrapper modules unless the task explicitly requires a reusable abstraction.

## Exception types

- `new_service_scope`: a newly introduced Azure service was not part of the original discovery set and needs one targeted metadata refresh.
- `missing_doc_uri`: the metadata entry does not include `documentationUri`, so targeted documentation lookup is allowed.
- `broken_doc_uri`: the `documentationUri` cannot be opened or is clearly stale or incomplete for the pinned version.
- `unresolved_registry_gap`: the initial metadata response did not contain the needed module, so one targeted follow-up verification is allowed.
- `producer_owned`: capability is exposed by the producer AVM and should be wired through that module.
- `exception_allowed_cycle`: a standalone native or AVM adjunct path is allowed because producer-owned wiring would create a real dependency cycle or require an unavailable runtime value.
- `exception_allowed_scope`: a standalone path is allowed because the target scope cannot be owned by the producer module.
- `unsupported_exception`: the authored fallback is not justified and must be removed.

## Review points

Watch for:

- local modules that only wrap a single AVM module
- native resources that should now be AVM modules
- cataloged producer modules that still declare sibling resources or helper modules for documented capabilities
- standalone role assignments, private endpoints, or diagnostics modules that the producer AVM can own
- native producer-owned extension resources that were not target-verified against the pinned producer AVM
- standalone AVM helper modules that duplicate producer-owned capabilities without a recorded cycle/scope exception
