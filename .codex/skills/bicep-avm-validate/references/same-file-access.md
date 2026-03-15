# Same-File Access

Use this reference to validate same-file access wiring between managed-identity consumer resources and Storage or Key Vault.

## Scope

Apply this rule only when the same authored file contains:

- a managed-identity consumer resource, and
- a Storage account or Key Vault producer resource that the consumer resource consumes

Covered managed-identity consumer modules include:

- Function App / Web App via `br/public:avm/res/web/site:<version>`
- Virtual Machine via `br/public:avm/res/compute/virtual-machine:<version>`
- Container App via `br/public:avm/res/app/container-app:<version>`
- Application Gateway via `br/public:avm/res/network/application-gateway:<version>`
- API Management via `br/public:avm/res/api-management/service:<version>`
- Any other module in-file with managed identity enabled and direct Storage/Key Vault consumption intent

Covered data producers include:

- Storage account via `br/public:avm/res/storage/storage-account:<version>`
- Key Vault vault via `br/public:avm/res/key-vault/vault:<version>`

## Required posture

- If the consumer resource is created first and the data resource depends on it or uses its outputs, require producer-owned `roleAssignments` on the Storage account or Key Vault in that same file.
- Treat consumer-only configuration such as app settings, URIs, or identity toggles as insufficient without the producer-owned access grant.
- Prefer producer-owned `roleAssignments` over standalone role-assignment resources or helper modules.
- Treat these as consumption intent signals that require same-file producer-owned access wiring:
  - certificate or TLS binding from Key Vault
  - secret or named-value reference from Key Vault
  - key operation usage (for example sign, wrap, unwrap) from Key Vault
  - storage data access through identity-based configuration

## Hard blockers

- Any managed-identity consumer authored alongside Storage or Key Vault in the same file, with consumption intent present, but the data resource has no producer-owned `roleAssignments`.
- Same-file Storage or Key Vault access that is deferred to standalone AVM/native role-assignment resources or helper modules instead of the producer module.
- Application Gateway or API Management declared as consuming Key Vault certificates or secrets in the same file, but Key Vault producer-owned `roleAssignments` are missing and no cycle/scope exception is documented.

## Reporting

- Report same-file Storage/Key Vault access posture as `pass` when producer-owned access is granted in the same file.
- Report posture as `blocker` when same-file consumer-to-data access is present but producer-owned `roleAssignments` are missing.
