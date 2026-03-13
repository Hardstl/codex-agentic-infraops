# Authoring Contract

Use this reference for the minimum contract every authored entrypoint must satisfy.

## Entrypoint contract

- Author one env-specific entrypoint for the requested scope.
- Use direct AVM registry module references for AVM-covered resources.
- Treat any AVM-covered native resource, local wrapper, or sibling AVM resource module as a blocker unless `references/avm-resolution.md` records a valid exception.
- Avoid local wrapper modules for AVM-covered resources.
- Use `references/producer-owned-capabilities.md` as the normative source for producer-owned capabilities.
- For cataloged producer modules, use the documented producer-owned parameters instead of sibling resources, child resources, or helper modules.
- Keep fixed design decisions in module logic and properties.
- Use parameters only for runtime, secret, or operational inputs.
- Use managed identity by default unless the task context requires another approach.
- If the authored file creates a managed-identity consumer and also creates a producer resource that consumer must access, wire the required data-plane access in that same authored file when the target scope and principal are known at authoring time.
- Do not declare `existing` resources for targets created in the same authored file.

## Baseline parameters

Use these when the task does not already define a stronger contract:

```bicep
@description('Location for all resources')
param location string

@description('Project name for tagging and naming')
param projectName string

@description('Environment for tagging and naming (e.g. dev, test, prod)')
param environment string
```

## Completion bar

Before handing off:

1. Resolve module choices and versions.
2. Check `references/producer-owned-capabilities.md` for every selected producer module that appears in the catalog.
3. Apply shared guardrails.
4. Run diagnostics.
5. Run `bicep build`.
6. Run `bicep lint`.
7. Fix blockers or record why a blocker remains.

## Anti-patterns

Do not:

- re-run research or service-selection logic
- introduce decision-proxy parameters for settled design choices
- create local AVM wrapper modules because they feel reusable
- keep sibling resources or helper modules for cataloged producer-owned capabilities when the producer module already owns them
- leave same-file managed-identity consumers without required access to same-file producer resources when producer-owned parameters or same-file role assignments can satisfy the dependency
- skip build or lint because diagnostics already passed
- mark inferred details as verified
