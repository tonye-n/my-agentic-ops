---
description: "Bicep coding standards for Azure infrastructure"
applyTo: "**/*.bicep"
---

# Bicep Development Standards

## Module Priority

1. Always check Azure Verified Modules (AVM) first.
   AVM reference format: `br/public:avm/res/<provider>/<resource>:<version>`
2. Use an AVM module when one exists for the resource type.
3. Fall back to a native `resource` block only when no AVM module is available.

## Security Defaults

Every generated Bicep file must:

- Set `httpsOnly: true` on all App Service resources.
- Set `minimumTlsVersion: '1.2'` on all web and storage resources.
- Assign a `SystemAssigned` managed identity to every App Service.
- Grant service identities access via RBAC `roleAssignments`. Never put connection strings or passwords in `appSettings`.

## Parameters and Outputs

- Declare `workloadName`, `environment`, and `location` as top-level parameters with `@description` decorators.
- Never hard-code subscription IDs, tenant IDs, resource group names, or passwords.
- Export at minimum: App Service default hostname and Key Vault URI.

## Structure

- Use a single `main.bicep` for deployments of six resources or fewer.
- Always pair `main.bicep` with a `main.bicepparam` file.