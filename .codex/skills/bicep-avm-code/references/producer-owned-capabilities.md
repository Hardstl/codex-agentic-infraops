# Producer-Owned Capabilities

Use this reference as the enforcement source for producer-owned AVM capabilities that commonly drift into sibling resources, helper modules, or native extension resources.

If a producer module appears here, use its listed parameters first. Do not author sibling resources or helper modules for the listed capabilities unless an allowed exception applies.

## Shared defaults

Apply these defaults to every producer module in this catalog when the parameter exists on that module:

- Shared must-use parameters:
  - `privateEndpoints`
  - `roleAssignments`
- Shared forbidden sibling shapes:
  - standalone private endpoint resources or modules attached to the same producer when `privateEndpoints` satisfies the requirement
  - standalone role-assignment resources or modules for that producer when `roleAssignments` satisfies the requirement

## Same-file access wiring

- When a managed-identity consumer and a producer resource are both authored in the same file, prefer producer-owned `roleAssignments` or equivalent producer-owned access parameters to grant required data-plane access.
- Require same-file access wiring when the consumer principal id and producer target are known at authoring time.
- Allow deferral only when a real `consumer -> producer -> consumer` dependency cycle or other runtime-only principal constraint prevents authoring-time wiring.

Repeat a shared capability inside a module section only when that module needs a narrower rule or a special exception note.

## Table of contents

- Key Vault vault
- SQL server
- Web site / Function App
- Virtual machine
- Virtual network
- Storage account
- Private DNS zone
- Container App
- App configuration store
- Application Gateway
- Data protection backup vault
- Redis cache
- Service Bus namespace

## Key Vault vault

- Producer module: `br/public:avm/res/key-vault/vault:<version>`
- Module-specific must-use parameters:
  - none
- Same-file access rule:
  - If a same-file managed-identity consumer needs secrets, keys, or certificates from the vault and the principal is known at authoring time, grant that data-plane access in the same file through the vault module's producer-owned `roleAssignments`.
- Forbidden sibling shapes:
  - standalone `Microsoft.Authorization/roleAssignments` or AVM/PTN role-assignment modules for vault access when the principal id is already known at authoring time
- Allowed exceptions:
  - `exception_allowed_cycle` when a system-assigned consumer identity would create a `consumer -> producer -> consumer` dependency cycle and the principal id is unavailable until deployment
  - `exception_allowed_scope` when the target scope cannot be owned by the vault module
- Minimal positive shape:

```bicep
module keyVault 'br/public:avm/res/key-vault/vault:0.11.0' = {
  name: 'mod-${kvName}'
  params: {
    name: kvName
    location: location
    enableRbacAuthorization: true
    roleAssignments: [
      {
        roleDefinitionIdOrName: kvSecretsUserRoleDefinitionId
        principalId: appPrincipalId
        principalType: 'ServicePrincipal'
      }
    ]
    privateEndpoints: [
      {
        name: '${kvName}-pe'
        subnetResourceId: privateEndpointSubnetResourceId
      }
    ]
  }
}
```

## SQL server

- Producer module: `br/public:avm/res/sql/server:<version>`
- Module-specific must-use parameters:
  - `databases`
- Forbidden sibling shapes:
  - separate SQL database resources or AVM database modules for databases that belong under the same SQL server deployment
- Allowed exceptions:
  - `exception_allowed_cycle` when producer-owned wiring would require a runtime-only principal id and no user-assigned identity or phased boundary is available
  - `exception_allowed_scope` when the target scope cannot be owned by the SQL server module
- Minimal positive shape:

```bicep
module sqlServer 'br/public:avm/res/sql/server:0.10.0' = {
  name: 'mod-${sqlServerName}'
  params: {
    name: sqlServerName
    location: location
    databases: [
      {
        name: sqlDatabaseName
      }
    ]
    privateEndpoints: [
      {
        name: '${sqlServerName}-pe'
        subnetResourceId: privateEndpointSubnetResourceId
      }
    ]
  }
}
```

## Web site / Function App

- Producer module: `br/public:avm/res/web/site:<version>`
- Module-specific must-use parameters:
  - `virtualNetworkSubnetResourceId`
  - `configs`
  - `managedIdentities`
- Same-file storage wiring rule:
  - When the same file creates the Function App and its backing storage account, derive `configs[*].storageAccountResourceId` from the storage account name with `resourceId('Microsoft.Storage/storageAccounts', functionStorageName)` to avoid dependency cycles.
- Same-file access note:
  - Identity-based app settings or site configuration only establish consumer-side usage. When the same file also creates the producer resource, still grant any separate producer-side data-plane access through that producer's `roleAssignments` or equivalent producer-owned access parameters.
- Forbidden sibling shapes:
  - separate VNet integration resources or child resources for regional VNet integration when `virtualNetworkSubnetResourceId` satisfies the requirement
  - standalone app settings or site-config resources when `configs` satisfies the requirement
  - separate identity-enablement resources or patterns that avoid `managedIdentities`
- Allowed exceptions:
  - `exception_allowed_cycle` when producer-owned role assignment would require a runtime-only system-assigned principal id and no user-assigned identity or phased boundary is available
  - `exception_allowed_scope` when the target scope cannot be owned by the site module
- Minimal positive shape:

```bicep
module functionApp 'br/public:avm/res/web/site:0.12.0' = {
  name: 'mod-${functionAppName}'
  params: {
    name: functionAppName
    kind: 'functionapp'
    serverFarmResourceId: appServicePlanResourceId
    virtualNetworkSubnetResourceId: integrationSubnetResourceId
    managedIdentities: {
      systemAssigned: true
    }
    configs: [
      {
        name: 'appsettings'
        storageAccountResourceId: resourceId('Microsoft.Storage/storageAccounts', functionStorageName)
        storageAccountUseIdentityAuthentication: true
      }
    ]
    privateEndpoints: [
      {
        name: '${functionAppName}-pe'
        subnetResourceId: privateEndpointSubnetResourceId
      }
    ]
  }
}
```

## Virtual machine

- Producer module: `br/public:avm/res/compute/virtual-machine:<version>`
- Module-specific must-use parameters:
  - `nicConfigurations`
  - `managedIdentities`
- Forbidden sibling shapes:
  - separate `Microsoft.Network/networkInterfaces` or `Microsoft.Network/publicIPAddresses` resources when `nicConfigurations` handles NIC and IP wiring for the VM.
  - standalone identity-only resources when the module can enable system-assigned/user-assigned identities via `managedIdentities`.
- Allowed exceptions:
  - `exception_allowed_scope` when required NICs must remain outside the module scope (for example, pre-existing network interfaces that cannot be reprovisioned in this deployment).
- Minimal positive shape:

```bicep
module virtualMachine 'br/public:avm/res/compute/virtual-machine:0.13.1' = {
  name: 'mod-${vmName}'
  params: {
    name: vmName
    location: location
    vmSize: vmSize
    osType: osType
    nicConfigurations: [
      {
        name: '${vmName}-nic'
        ipConfigurations: [
          {
            name: 'ipconfig01'
            subnetResourceId: subnetResourceId
          }
        ]
      }
    ]
    managedIdentities: {
      systemAssigned: true
      userAssignedResourceIds: [
        userAssignedIdentityResourceId
      ]
    }
    roleAssignments: [
      {
        roleDefinitionIdOrName: vmIdentityRoleDefinitionId
        principalId: vmIdentityPrincipalId
      }
    ]
}
}
```

## Virtual network

- Producer module: `br/public:avm/res/network/virtual-network:<version>`
- Module-specific must-use parameters:
  - `addressPrefixes`
  - `subnets`
  - `peerings`
  - `diagnosticSettings`
  - `lock`
- Forbidden sibling shapes:
  - standalone `Microsoft.Network/virtualNetworks/subnets`, `/virtualNetworkPeerings`, diagnostics, or lock resources when the module manages them via `subnets`, `peerings`, `diagnosticSettings`, and `lock`
  - separate `Microsoft.Authorization/roleAssignments` or AVM role-assignment modules when the shared `roleAssignments` parameter covers the need
  - distinct virtual network resources (local wrappers) when `br/public:avm/res/network/virtual-network` already covers the required configuration
- Allowed exceptions:
  - `exception_allowed_scope` when remote peering or subnet provisioning must happen in another scope that the module cannot reach
- Minimal positive shape:

```bicep
module virtualNetwork 'br/public:avm/res/network/virtual-network:0.7.0' = {
  name: 'mod-${vnetName}'
  params: {
    name: vnetName
    location: location
    addressPrefixes: [
      '10.0.0.0/16'
    ]
    subnets: [
      {
        name: 'app-subnet'
        addressPrefix: '10.0.1.0/24'
        roleAssignments: [
          {
            roleDefinitionIdOrName: subnetContributorRoleId
            principalId: subnetPrincipalId
          }
        ]
      }
    ]
    peerings: [
      {
        remoteVirtualNetworkResourceId: otherVnetId
        allowVirtualNetworkAccess: true
        useRemoteGateways: true
      }
    ]
    diagnosticSettings: [
      {
        name: '${vnetName}-diag'
        workspaceId: logWorkspaceId
      }
    ]
    lock: {
      kind: 'CanNotDelete'
    }
    roleAssignments: [
      {
        roleDefinitionIdOrName: vnetContributorRoleId
        principalId: vnetPrincipalId
      }
    ]
}
}
```

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
  - `lock`
- Same-file access rule:
  - If a same-file managed-identity consumer needs blob, file, queue, or table data-plane access and the principal is known at authoring time, grant that access in the same file through the storage account module's producer-owned `roleAssignments`.
- Forbidden sibling shapes:
  - separate `Microsoft.Storage/storageAccounts/blobServices`, `fileServices`, `queueServices`, `tableServices`, `managementPolicies`, `localUsers`, or `objectReplicationPolicies` resources when their configuration is already captured via the producer parameters
  - standalone `Microsoft.Network/privateEndpoints` or AVM private-endpoint modules when `privateEndpoints` handles service connectivity
  - separate key vault secrets export, diagnostics, or lock resources when the module parameters already produce them
- Allowed exceptions:
  - `exception_allowed_cycle` when the role assignment requires a principal id only available at runtime and cannot be resolved through system/user assigned identities
  - `exception_allowed_scope` when the child resource must exist in an external scope (for example, preexisting private endpoints or replication policies)
- Minimal positive shape:

```bicep
module storageAccount 'br/public:avm/res/storage/storage-account:0.18.0' = {
  name: 'mod-${storageName}'
  params: {
    name: storageName
    location: location
    kind: 'StorageV2'
    skuName: 'Standard_GRS'
    allowBlobPublicAccess: false
    minimumTlsVersion: 'TLS1_2'
    supportsHttpsTrafficOnly: true
    networkAcls: {
      defaultAction: 'Deny'
      virtualNetworkRules: [
        {
          id: subnetResourceId
        }
      ]
    }
    privateEndpoints: [
      {
        subnetResourceId: subnetResourceId
      }
    ]
    diagnosticSettings: [
      {
        name: '${storageName}-diag'
        workspaceId: logWorkspaceId
      }
    ]
    lock: {
      kind: 'CanNotDelete'
    }
    roleAssignments: [
      {
        roleDefinitionIdOrName: storageContributorRoleId
        principalId: storagePrincipalId
      }
    ]
  }
}
```

## Private DNS zone

- Producer module: `br/public:avm/res/network/private-dns-zone:<version>`
- Module-specific must-use parameters:
  - `virtualNetworkLinks`
  - `lock`
  - `roleAssignments`
- Forbidden sibling shapes:
  - separate `Microsoft.Network/privateDnsZones/*` child resources for records, links, diagnostics, locks, or role assignments when the module parameters already express them
- Allowed exceptions:
  - `exception_allowed_scope` when the zone must be linked to a vNet or record in another subscription or deployment
- Minimal positive shape:

```bicep
module privateDnsZone 'br/public:avm/res/network/private-dns-zone:0.4.0' = {
  name: 'mod-${dnsZoneName}'
  params: {
    name: dnsZoneName
    location: 'global'
    virtualNetworkLinks: [
      {
        virtualNetworkResourceId: virtualNetworkId
        registrationEnabled: true
      }
    ]
    lock: {
      kind: 'CanNotDelete'
    }
    roleAssignments: [
      {
        roleDefinitionIdOrName: 'Private DNS Zone Contributor'
        principalId: dnsAdminPrincipalId
      }
    ]
  }
}
```

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
- Allowed exceptions:
  - `exception_allowed_scope` when the child resource must be defined in another deployment or subscription (for hosted ingress edge routers, pre-existing identities, etc.)
- Minimal positive shape:

```bicep
module containerApp 'br/public:avm/res/app/container-app:0.5.0' = {
  name: 'mod-${containerAppName}'
  params: {
    name: containerAppName
    location: location
    environmentResourceId: containerAppEnvironmentId
    containers: [
      {
        name: 'api'
        image: containerImage
        resources: {
          cpu: 1
          memory: '2Gi'
        }
      }
    ]
    service: {
      ingress: {
        external: true
        targetPort: 8080
      }
    }
    identitySettings: {
      type: 'SystemAssigned'
    }
    managedIdentities: {
      systemAssigned: true
    }
    roleAssignments: [
      {
        roleDefinitionIdOrName: containerAppContributorRoleId
        principalId: containerAppPrincipalId
      }
    ]
    diagnosticSettings: [
      {
        name: '${containerAppName}-diag'
        workspaceId: logWorkspaceId
      }
    ]
  }
}
```

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
- Allowed exceptions:
  - `exception_allowed_scope` when an existing backend pool, certificate, listener, or path map must remain in another subscription or deployment
  - `exception_allowed_cycle` when gateway role assignments or private endpoint wiring depend on a principal id that only resolves at runtime
- Minimal positive shape:

```bicep
module appGateway 'br/public:avm/res/network/application-gateway:<version>' = {
  name: 'mod-${appGatewayName}'
  params: {
    name: appGatewayName
    location: location
    backendAddressPools: [
      {
        name: 'appgw-backend'
        backendAddresses: [
          {
            fqdn: 'backend.internal'
          }
        ]
      }
    ]
    backendHttpSettingsCollection: [
      {
        name: 'appgw-http'
        port: 443
        protocol: 'Https'
      }
    ]
    listeners: [
      {
        name: 'appgw-listener'
        frontendPort: 'appgw-frontend-port'
        sslCertificate: 'appgw-ssl'
      }
    ]
    requestRoutingRules: [
      {
        name: 'appgw-rule'
        ruleType: 'Basic'
        httpListener: 'appgw-listener'
        backendAddressPool: 'appgw-backend'
        backendHttpSettings: 'appgw-http'
      }
    ]
    urlPathMaps: [
      {
        name: 'appgw-path-map'
        defaultBackendAddressPool: 'appgw-backend'
        defaultBackendHttpSettings: 'appgw-http'
      }
    ]
    sslCertificates: [
      {
        name: 'appgw-ssl'
        data: sslCertificateData
        password: sslCertificatePassword
      }
    ]
    privateLinkConfigurations: [
      {
        name: 'appgw-plc'
        subnetResourceId: privateLinkSubnetResourceId
      }
    ]
    privateEndpoints: [
      {
        name: 'appgw-pe'
        subnetResourceId: privateEndpointSubnetResourceId
      }
    ]
    roleAssignments: [
      {
        roleDefinitionIdOrName: networkContributorRoleDefinitionId
        principalId: appGatewayIdentityPrincipalId
      }
    ]
  }
}
```

## Data protection backup vault

- Producer module: `br/public:avm/res/data-protection/backup-vault:<version>`
- Module-specific must-use parameters:
  - `backupPolicies`
  - `backupInstances`
- Forbidden sibling shapes:
  - `Microsoft.DataProtection/backupVaults/backupPolicies` or helper modules for policies when `backupPolicies` is supplied
  - `Microsoft.DataProtection/backupVaults/backupInstances` or helper modules for instances when `backupInstances` is supplied
- Allowed exceptions:
  - `exception_allowed_scope` when the policy or backup instance must live in a different scope that the vault module cannot reach
  - `exception_allowed_cycle` when the vault needs a runtime-only principal id to deploy a policy or instance
- Minimal positive shape:

```bicep
module backupVault 'br/public:avm/res/data-protection/backup-vault:<version>' = {
  name: 'mod-${backupVaultName}'
  params: {
    name: backupVaultName
    location: location
    backupPolicies: [
      {
        name: 'daily-retention'
        properties: {
          backupManagementType: 'AzureIaasVM'
          schedulePolicy: {
            scheduleRunFrequency: 'Daily'
            scheduleRunTimes: [
              '2026-03-10T00:00:00Z'
            ]
          }
        }
      }
    ]
    backupInstances: [
      {
        name: 'vm-backup-instance'
        dataSourceInfo: backupInstanceDataSource
        policyInfo: backupInstancePolicy
      }
    ]
    privateEndpoints: [
      {
        name: '${backupVaultName}-pe'
        subnetResourceId: privateEndpointSubnetResourceId
      }
    ]
    roleAssignments: [
      {
        roleDefinitionIdOrName: backupContributorRoleDefinitionId
        principalId: backupOperatorPrincipalId
      }
    ]
  }
}
```

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
- Allowed exceptions:
  - `exception_allowed_scope` when the firewall rule, linked server, or secret export must be owned by another deployment
  - `exception_allowed_cycle` when access policy assignments require a principal id that resolves only at runtime
- Minimal positive shape:

```bicep
module redisCache 'br/public:avm/res/cache/redis:<version>' = {
  name: 'mod-${redisName}'
  params: {
    name: redisName
    location: location
    skuName: 'Premium'
    accessPolicies: [
      {
        name: 'default-policy'
        permissions: 'default'
      }
    ]
    accessPolicyAssignments: [
      {
        name: 'default-assignment'
        objectId: assignmentObjectId
        accessPolicyName: 'default-policy'
      }
    ]
    firewallRules: [
      {
        name: 'trusted-subnet'
        startIP: '10.0.0.0'
        endIP: '10.0.0.255'
      }
    ]
    geoReplicationObject: {
      linkedRedisCacheResourceId: geoSecondaryRedisId
    }
    secretsExportConfiguration: {
      keyVaultResourceId: secretsKeyVaultId
      primaryAccessKeyName: 'redis-primary-key'
    }
    privateEndpoints: [
      {
        name: '${redisName}-pe'
        subnetResourceId: privateEndpointSubnetResourceId
      }
    ]
    roleAssignments: [
      {
        roleDefinitionIdOrName: redisContributorRoleDefinitionId
        principalId: redisPrincipalId
      }
    ]
  }
}
```

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
- Allowed exceptions:
  - `exception_allowed_scope` when the predicate resource must be landed in another subscription or deployment
  - `exception_allowed_cycle` when an authorization rule or queue/topic role assignment needs a principal id only available at runtime
- Minimal positive shape:

```bicep
module serviceBusNamespace 'br/public:avm/res/service-bus/namespace:<version>' = {
  name: 'mod-${serviceBusNamespaceName}'
  params: {
    name: serviceBusNamespaceName
    location: location
    authorizationRules: [
      {
        name: 'default'
        rights: [
          'Listen'
          'Manage'
          'Send'
        ]
      }
    ]
    queues: [
      {
        name: 'work-queue'
      }
    ]
    topics: [
      {
        name: 'events'
      }
    ]
    networkRuleSets: {
      virtualNetworkRules: [
        {
          subnetResourceId: privateEndpointSubnetResourceId
          ignoreMissingVnetServiceEndpoint: false
        }
      ]
    }
    disasterRecoveryConfig: {
      name: 'geo-pair'
      partnerNamespace: recoveryPartnerNamespaceId
    }
    migrationConfiguration: {
      postMigrationName: 'post-migration'
      targetNamespace: migrationTargetNamespaceId
    }
    privateEndpoints: [
      {
        name: '${serviceBusNamespaceName}-pe'
        subnetResourceId: privateEndpointSubnetResourceId
      }
    ]
    roleAssignments: [
      {
        roleDefinitionIdOrName: serviceBusContributorRoleDefinitionId
        principalId: serviceBusPrincipalId
      }
    ]
  }
}
```

## Catalog policy

- Treat a forbidden sibling shape for a cataloged module as a hard failure, not a preference miss.
- If a producer module is not listed here, still use the direct AVM producer module when coverage exists.
- If a producer module is not listed here, do not invent new producer-owned capability rules or new fallback sibling resources from memory.
- Use targeted upstream proof only to justify a concrete exception path already present in authored code or required by a validator blocker.
