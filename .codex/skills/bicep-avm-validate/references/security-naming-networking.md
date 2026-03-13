# Security, Naming, and Networking

Use this reference to validate shared guardrails across authored Bicep.

## Tags

Require these tags where the resource type supports tags:

- `Environment`
- `ManagedBy`
- `Project`

## Security defaults

- Prefer managed identity over secrets, keys, and connection strings.
- Require `minimumTlsVersion: 'TLS1_2'` where the producer supports it.
- Storage defaults:
  - `supportsHttpsTrafficOnly: true`
  - `allowBlobPublicAccess: false`
  - `allowSharedKeyAccess: false`
- Key Vault default:
  - `enableRbacAuthorization: true`
- When private endpoints are used, require public network access to be disabled unless authored code clearly documents a mixed-access intent.

## Naming

- Use a deterministic naming pattern based on type, project, and environment.
- Put the type token first in top-level resource names.
- Keep names lowercase when the Azure resource type requires lowercase.
- Storage account names must be lowercase alphanumeric only.
- NSG names should be derived from the subnet name with a type-first prefix such as `nsg-<subnetNameToken>`.

## Networking and exposure

- All non-reserved subnets must include NSG association.
- All non-reserved subnets should set `privateEndpointNetworkPolicies: 'Enabled'`.
- Reserved subnet exemptions are limited to:
  - `AzureFirewallSubnet`
  - `AzureFirewallManagementSubnet`
  - `GatewaySubnet`
  - `RouteServerSubnet`
- Prefer producer-owned private endpoints when the AVM supports them.
- When authored intent is clearly private, corp-only, or no-public-access, require private endpoint wiring and treat public exposure as a blocker unless mixed access is explicitly called for in the task context.

## Reporting

- Report security, naming, and networking posture as `pass` when these guardrails are met.
- Report posture as `blocker` when tags, defaults, naming constraints, subnet guardrails, or exposure rules are violated.
