# Producer-Owned Capabilities

Use this reference as the enforcement source for producer-owned AVM capabilities that commonly drift into sibling resources, helper modules, or native extension resources.

If a producer module appears here, validate its listed parameters first. Report a blocker when authored code keeps sibling resources or helper modules for the listed capabilities.

## Shared defaults

Apply these defaults to every producer module in this catalog when the parameter exists on that module:

- Shared must-use parameters:
  - `privateEndpoints`
  - `roleAssignments`
- Shared forbidden sibling shapes:
  - standalone private endpoint resources or modules attached to the same producer when `privateEndpoints` satisfies the requirement
  - standalone role-assignment resources or modules for that producer when `roleAssignments` satisfies the requirement
- Shared hard-fail module shapes:
  - standalone AVM `roleAssignments` modules
  - standalone AVM `privateEndpoints` modules

## Same-file access wiring

- When a managed-identity compute consumer and a Storage account or Key Vault producer are both authored in the same file, require producer-owned `roleAssignments` on the data producer.
- Treat consumer-only settings such as URIs, app settings, or identity toggles as insufficient without the producer-owned access grant.
- When the compute resource is created first and the data producer depends on it or uses its outputs, treat same-file access wiring as required.

Repeat a shared capability inside a module section only when that module needs a narrower rule or a special validator note.

## Table of contents

- Key Vault vault
- SQL server
- Web site / Function App
- Virtual machine
- Virtual network
- Storage account
- Private DNS zone
- Container App
- Application Gateway
- Data protection backup vault
- Redis cache
- Service Bus namespace

## Key Vault vault

- Producer module: `br/public:avm/res/key-vault/vault:<version>`
- Module-specific must-use parameters:
  - `roleAssignments` for same-file compute access
- Forbidden sibling shapes:
  - standalone `Microsoft.Authorization/roleAssignments` or AVM/PTN role-assignment modules for vault access
  - standalone `Microsoft.Network/privateEndpoints` or AVM private-endpoint modules when the vault module uses `privateEndpoints`
- Same-file access rule:
  - If a same-file managed-identity compute resource consumes the vault, require vault-module `roleAssignments` in that same file.

## SQL server

- Producer module: `br/public:avm/res/sql/server:<version>`
- Module-specific must-use parameters:
  - `databases`
- Forbidden sibling shapes:
  - separate SQL database resources or AVM database modules for databases that belong under the same SQL server deployment

## Web site / Function App

- Producer module: `br/public:avm/res/web/site:<version>`
- Module-specific must-use parameters:
  - `virtualNetworkSubnetResourceId`
  - `configs`
  - `managedIdentities`
- Same-file storage wiring rule:
  - When the same file creates the Function App and its backing storage account, derive `configs[*].storageAccountResourceId` from the storage account name with `resourceId('Microsoft.Storage/storageAccounts', functionStorageName)` to avoid dependency cycles.
- Same-file access note:
  - Identity-based app settings or site configuration only establish consumer-side usage. When the same file also creates Storage or Key Vault, still require producer-owned `roleAssignments` on that data producer.
- Forbidden sibling shapes:
  - separate VNet integration resources or child resources for regional VNet integration when `virtualNetworkSubnetResourceId` satisfies the requirement
  - standalone app settings or site-config resources when `configs` satisfies the requirement
  - separate identity-enablement resources or patterns that avoid `managedIdentities`

## Virtual machine

- Producer module: `br/public:avm/res/compute/virtual-machine:<version>`
- Module-specific must-use parameters:
  - `nicConfigurations`
  - `managedIdentities`
- Forbidden sibling shapes:
  - separate `Microsoft.Network/networkInterfaces` or `Microsoft.Network/publicIPAddresses` resources when `nicConfigurations` handles NIC and IP wiring for the VM
  - standalone identity-only resources when the module can enable system-assigned or user-assigned identities via `managedIdentities`

## Virtual network

- Producer module: `br/public:avm/res/network/virtual-network:<version>`
- Module-specific must-use parameters:
  - `addressPrefixes`
  - `subnets`
  - `peerings`
  - `diagnosticSettings`
- Forbidden sibling shapes:
  - standalone `Microsoft.Network/virtualNetworks/subnets`, `/virtualNetworkPeerings`, diagnostics, or lock resources when the module manages them via `subnets`, `peerings`, `diagnosticSettings`, and `lock`
  - separate `Microsoft.Authorization/roleAssignments` or AVM role-assignment modules when the shared `roleAssignments` parameter covers the need
  - distinct virtual network resources when the AVM virtual network module already covers the required configuration

## Storage account

- Producer module: `br/public:avm/res/storage/storage-account:<version>`
- Module-specific must-use parameters:
  - `kind`
  - `skuName`
  - `networkAcls`
  - `allowBlobPublicAccess`
  - `minimumTlsVersion`
  - `supportsHttpsTrafficOnly`
  - `diagnosticSettings`
  - `roleAssignments` for same-file compute access
- Same-file access rule:
  - If a same-file managed-identity compute resource consumes blob, file, queue, or table data and the compute resource is created first with the storage account depending on it or using its outputs, require storage-account-module `roleAssignments` in that same file.
- Forbidden sibling shapes:
  - separate `Microsoft.Storage/storageAccounts/blobServices`, `fileServices`, `queueServices`, `tableServices`, `managementPolicies`, `localUsers`, or `objectReplicationPolicies` resources when their configuration is already captured via the producer parameters
  - standalone `Microsoft.Network/privateEndpoints` or AVM private-endpoint modules when `privateEndpoints` handles service connectivity
  - standalone role-assignment resources or helper modules for storage data access
  - separate diagnostics or lock resources when the module parameters already produce them

## Private DNS zone

- Producer module: `br/public:avm/res/network/private-dns-zone:<version>`
- Module-specific must-use parameters:
  - `virtualNetworkLinks`
  - `roleAssignments`
- Forbidden sibling shapes:
  - separate `Microsoft.Network/privateDnsZones/*` child resources for records, links, diagnostics, locks, or role assignments when the module parameters already express them

## Container App

- Producer module: `br/public:avm/res/app/container-app:<version>`
- Module-specific must-use parameters:
  - `containers`
  - `service`
  - `identitySettings`
  - `diagnosticSettings`
- Forbidden sibling shapes:
  - separate resource blocks for `Microsoft.App/containerApps` children such as container definitions, service bindings, or ingress configuration when the producer parameters already express them
  - standalone identity/configuration resources when `managedIdentities`, `identitySettings`, and `roleAssignments` satisfy the needs

## Application Gateway

- Producer module: `br/public:avm/res/network/application-gateway:<version>`
- Module-specific must-use parameters:
  - `backendAddressPools`
  - `backendHttpSettingsCollection`
  - `listeners`
  - `requestRoutingRules`
  - `urlPathMaps`
  - `sslCertificates`
  - `privateLinkConfigurations`
- Forbidden sibling shapes:
  - child resources for backend pools, HTTP settings, listeners, routing rules, URL path maps, probes, or SSL certificates when the corresponding module parameters express the same configuration
  - `Microsoft.Network/applicationGateways/privateLinkConfigurations` or standalone AVM private endpoints when the gateway already describes them via `privateLinkConfigurations` and `privateEndpoints`

## Data protection backup vault

- Producer module: `br/public:avm/res/data-protection/backup-vault:<version>`
- Module-specific must-use parameters:
  - `backupPolicies`
  - `backupInstances`
- Forbidden sibling shapes:
  - `Microsoft.DataProtection/backupVaults/backupPolicies` or helper modules for policies when `backupPolicies` is supplied
  - `Microsoft.DataProtection/backupVaults/backupInstances` or helper modules for instances when `backupInstances` is supplied

## Redis cache

- Producer module: `br/public:avm/res/cache/redis:<version>`
- Module-specific must-use parameters:
  - `accessPolicies`
  - `accessPolicyAssignments`
  - `firewallRules`
  - `geoReplicationObject`
  - `secretsExportConfiguration`
- Forbidden sibling shapes:
  - `Microsoft.Cache/redis/firewallRules`, `accessPolicies`, or `accessPolicyAssignments` resources when the module parameters already deploy them
  - standalone `Microsoft.Cache/redis/linkedServers` resources when `geoReplicationObject` configures the geo-replication pair
  - helper modules that export secrets or wire Key Vault references when `secretsExportConfiguration` already covers the need

## Service Bus namespace

- Producer module: `br/public:avm/res/service-bus/namespace:<version>`
- Module-specific must-use parameters:
  - `authorizationRules`
  - `queues`
  - `topics`
  - `networkRuleSets`
  - `disasterRecoveryConfig`
  - `migrationConfiguration`
- Forbidden sibling shapes:
  - separate `Microsoft.ServiceBus/namespaces/authorizationRules`, `queues`, `topics`, `networkRuleSets`, `disasterRecoveryConfigs`, or `migrationConfigurations` when the module parameters already declare them

## Catalog policy

- Treat a forbidden sibling shape for a cataloged module as a hard failure, not a preference miss.
- Treat standalone AVM `roleAssignments` or `privateEndpoints` modules as hard failures in all cases.
- If a producer module is not listed here, still prefer the direct AVM producer module when coverage exists.
- Do not invent new producer-owned capability rules for unlisted modules from memory; validate only against the local catalog and obvious standalone AVM `roleAssignments` or `privateEndpoints` module misuse.
