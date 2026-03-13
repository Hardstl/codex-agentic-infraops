# Security, Naming, and Networking

Use this reference for shared guardrails that apply across authored Bicep.

## Tags

Apply these tags where supported:

```bicep
var tags = {
  Environment: environment
  ManagedBy: 'Codex'
  Project: projectName
}
```

Required tags:

- `Environment`
- `ManagedBy`
- `Project`

## Security defaults

- Use `minimumTlsVersion: 'TLS1_2'` where supported.
- Prefer managed identity over secrets, keys, and connection strings.
- Storage defaults:
  - `supportsHttpsTrafficOnly: true`
  - `allowBlobPublicAccess: false`
  - `allowSharedKeyAccess: false`
- Key Vault default:
  - `enableRbacAuthorization: true`
- When private endpoints are used, disable public network access for supporting resources unless the task explicitly approves mixed access.

## Naming

Generate and reuse a deterministic suffix when uniqueness is required:

```bicep
var uniqueSuffix = uniqueString(resourceGroup().id)
```

Examples:

```bicep
var kvName = toLower('kv-${take(projectName, 11)}-${take(environment, 4)}-${take(uniqueSuffix, 4)}')
var storageName = toLower('st${take(replace(projectName, '-', ''), 14)}${take(environment, 4)}${take(uniqueSuffix, 4)}')
```

Rules:

- Use `<type-token>-<project-token>-<environment-token>[-<uniqueSuffix>]` for top-level resource names.
- Put the type token first.
- Append uniqueness only when the Azure resource type requires it.
- Keep names lowercase when the resource type requires lowercase.
- Storage account names must be lowercase alphanumeric only.
- Name NSGs from the subnet name with a type-first prefix such as `nsg-<subnetNameToken>`.

## Networking and exposure

- All non-reserved subnets must include NSG association.
- All non-reserved subnets should set `privateEndpointNetworkPolicies: 'Enabled'`.
- Reserved subnet exemptions are limited to:
  - `AzureFirewallSubnet`
  - `AzureFirewallManagementSubnet`
  - `GatewaySubnet`
  - `RouteServerSubnet`
- Prefer producer-owned private endpoints when the AVM supports them.
- When the brief expresses private intent (keywords such as “private,” “corp,” “no public access,” etc.), treat that as a mandate to wire producer-owned `privateEndpoints`/Private DNS for every resource that can consume them and to disable `publicNetworkAccess` unless a documented exception allows mixed exposure.
