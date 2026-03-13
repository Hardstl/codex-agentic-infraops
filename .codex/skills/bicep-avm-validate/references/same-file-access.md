# Same-File Access

Use this reference to validate same-file access wiring between compute resources and Storage or Key Vault.

## Scope

Apply this rule only when the same authored file contains:

- a compute resource with managed identity enabled, and
- a Storage account or Key Vault producer resource that the compute resource consumes

Covered compute producers include:

- Function App / Web App via `br/public:avm/res/web/site:<version>`
- Virtual Machine via `br/public:avm/res/compute/virtual-machine:<version>`
- Container App via `br/public:avm/res/app/container-app:<version>`

Covered data producers include:

- Storage account via `br/public:avm/res/storage/storage-account:<version>`
- Key Vault vault via `br/public:avm/res/key-vault/vault:<version>`

## Required posture

- If the compute resource is created first and the data resource depends on it or uses its outputs, require producer-owned `roleAssignments` on the Storage account or Key Vault in that same file.
- Treat consumer-only configuration such as app settings, URIs, or identity toggles as insufficient without the producer-owned access grant.
- Prefer producer-owned `roleAssignments` over standalone role-assignment resources or helper modules.

## Hard blockers

- Function App, Web App, Virtual Machine, or Container App authored with managed identity alongside Storage or Key Vault in the same file, but the data resource has no producer-owned `roleAssignments`.
- Same-file Storage or Key Vault access that is deferred to standalone AVM/native role-assignment resources or helper modules instead of the producer module.

## Reporting

- Report same-file Storage/Key Vault access posture as `pass` when producer-owned access is granted in the same file.
- Report posture as `blocker` when same-file compute-to-data access is present but producer-owned `roleAssignments` are missing.
