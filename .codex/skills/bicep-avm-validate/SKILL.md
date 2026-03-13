---
name: bicep-avm-validate
description: Validate authored Azure Bicep against AVM-first rules and report blockers with diagnostics, build, lint, and policy checks. Use when Codex needs validation-only review or needs to gate authored Bicep before sign-off. Do not use for research or infrastructure authoring.
---

# Bicep AVM Validate

Review authored Bicep without mutating the infrastructure design. This skill is a self-contained validator for AVM-first Azure Bicep.

## Prerequisite check

Before validation:

```powershell
bicep --version
```

Required tools:

- `mcp__bicep__get_bicep_file_diagnostics`
- `bicep build <entrypoint.bicep>`
- `bicep lint <entrypoint.bicep>`

## Workflow

1. Identify the env-specific entrypoint and supporting Bicep files in scope.
2. Run diagnostics, build, and lint before making semantic judgments.
3. Validate AVM-first posture with `references/avm-posture.md`.
4. Validate producer-owned capability usage with `references/producer-owned-capabilities.md`.
5. Validate security, naming, and networking guardrails with `references/security-naming-networking.md`.
6. Validate same-file Storage and Key Vault access wiring with `references/same-file-access.md`.
7. Report concrete blockers tied to the code, not generic advice.
8. Distinguish hard failures from optional improvements.

## Hard rules

- Do not author or modify infrastructure code as part of this skill.
- Do not start or continue a remediation loop from this skill unless the user explicitly asks for authoring or fixes.
- Do not collect user input.
- Do not run research workflows or service-selection lookups.
- Validate only what is present in the authored Bicep and local task context.
- Fail when the code violates the validator contract, even if the template compiles.
- Fail AVM-covered native resources, local wrappers, and sibling AVM resource/helper modules unless the validator-local contract explicitly allows the shape.
- Fail producer-owned capability drift when the code creates sibling resources, native extension resources, or helper modules that the producer AVM should own.
- Fail any standalone AVM module for `roleAssignments` or `privateEndpoints` as a hard blocker with no exception path; those must stay in the producer module.
- Fail security, naming, and networking violations from the validator-local guardrails, including insecure defaults, exposure drift, missing required tags, and subnet/NSG policy violations.
- Fail same-file compute-to-data wiring when a same-file compute resource and Storage or Key Vault resource are authored together but the data resource does not grant producer-owned `roleAssignments`.

## Reporting rules

- Report blockers as specific file-and-behavior failures.
- Treat AVM posture, producer-owned capability, security, naming, networking, identity, and dependency violations as blockers.
- Mark optional improvements separately from blockers.
- Report AVM posture explicitly as `pass` or `blocker`.
- Report producer-owned capability posture explicitly as `pass` or `blocker`.
- Report security, naming, and networking posture explicitly as `pass` or `blocker`.
- Report same-file Storage/Key Vault access posture explicitly as `pass` or `blocker`.
- Always report standalone AVM `roleAssignments` or `privateEndpoints` modules as blockers.
- Do not mark inferred details as verified.
