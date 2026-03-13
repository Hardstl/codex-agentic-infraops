# Remediation

Use this reference when validator blockers or local authoring failures need targeted fixes.

## Default remediation loop

1. Read the blocker text and identify the exact file and behavior.
2. Fix only the blocker and tightly related fallout.
3. Re-run diagnostics.
4. Re-run `bicep build`.
5. Re-run `bicep lint`.
6. Refresh the work status or task notes with the new result.

## Common blocker classes

### AVM posture

- Replace AVM-coverable native resources with direct AVM modules.
- Treat AVM-covered native resources, local wrappers, and sibling AVM resource modules as blockers until removed or justified by a recorded exception.
- Remove local wrapper modules that only proxy AVM parameters.
- Move cataloged producer-owned extensions and standalone AVM helper modules into the producer module first.
- If the capability is not cataloged, require targeted proof before keeping any producer-owned fallback path.
- If targeted proof cannot be completed, treat the exception path as a blocker rather than an acceptable workaround.

### Dependency cycles

- Remove unnecessary consumer dependencies on producer outputs.
- Derive names, IDs, and URIs when conventions make that safe.
- If needed, introduce a user-assigned identity or a phased deployment boundary.
- If neither is chosen, record the cycle reason before keeping a standalone helper/native exception path.

### Security defaults

- Re-enable managed identity where secrets or connection strings were introduced.
- Restore secure storage defaults.
- Disable public network access when private endpoint-only access was intended.

### Contract drift

- Remove decision-proxy parameters.
- Add or repair parameter descriptions in entrypoints.
- Remove `existing` declarations for resources created in the same file.
- Remove sibling resources or helper modules for cataloged producer-owned capabilities.

## Reporting

- Report blockers as concrete file-and-behavior issues.
- Distinguish hard failures from optional improvements.
- Report catalog violations as hard failures, not preferences.
- State whether each producer-owned capability case was `producer_owned`, `exception_allowed_cycle`, `exception_allowed_scope`, or `unsupported_exception`.
- State whether targeted proof was verified or failed closed because upstream proof was unavailable.
- Do not downgrade a contract violation to a suggestion.
