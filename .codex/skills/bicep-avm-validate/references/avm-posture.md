# AVM Posture

Use this reference to validate that authored Bicep follows an AVM-first posture.

## Required posture

- Use direct `br/public:avm/...` module references for AVM-covered Azure resources.
- Pin explicit AVM versions in authored files.
- Keep the producer resource in the direct AVM module rather than replacing it with native resources or local wrappers.

## Hard blockers

- AVM-covered native resources used instead of a direct AVM module.
- Local modules that only wrap or proxy a single AVM module without adding real composition that the validator contract requires.
- Sibling AVM resource or helper modules used where the direct producer AVM should own the resource shape.
- Mixed posture where some resources in the same covered service family are authored as direct AVM modules and others stay as native resources without a validator-local exception.

## Review points

Watch for:

- `resource` blocks for Storage, Key Vault, Web Site / Function App, Virtual Network, SQL Server, Virtual Machine, Service Bus, Redis, or other services already represented as `br/public:avm/...` modules in the authored file set
- local module paths that only pass parameters through to one AVM module
- AVM helper modules that exist only to land adjunct capabilities instead of configuring the producer module directly

## Reporting

- Report posture as `pass` when all AVM-covered resources are direct AVM modules.
- Report posture as `blocker` when any AVM-covered native resource, local wrapper, or sibling AVM resource/helper module remains.
