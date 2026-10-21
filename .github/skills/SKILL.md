---
name: azure-ops-defaults
description: "Shared defaults for Azure infrastructure provisioning"
---

# Azure Operations Defaults

## Naming Conventions

Follow Azure Cloud Adoption Framework (CAF) naming patterns.

Keep workload names 8 characters or fewer. Use `dev`, `test`, or `prod` for environment.

| Resource | Pattern | Example |
|---|---|---|
| Resource group | `rg-{workload}-{env}` | `rg-webapp-dev` |
| App Service plan | `asp-{workload}-{env}` | `asp-webapp-dev` |
| App Service | `app-{workload}-{env}` | `app-webapp-dev` |
| SQL Server | `sql-{workload}-{env}` | `sql-webapp-dev` |
| SQL Database | `sqldb-{workload}-{env}` | `sqldb-webapp-dev` |
| Key Vault | `kv-{workload}-{env}` | `kv-webapp-dev` |

## Security Baseline

Apply these rules to every deployment:

- **TLS**: Minimum TLS 1.2 on all public endpoints.
- **HTTPS**: Set `httpsOnly: true` on every App Service.
- **Authentication**: Use managed identity for service-to-service access.   Never store passwords or connection strings in application settings.
- **Key Vault access**: Use RBAC (`Key Vault Secrets User` role), not access policies.
- **SQL**: Enable Entra ID authentication alongside SQL authentication.

## Default SKUs

| Service | Development | Production |
|---|---|---|
| App Service plan | B1 (Linux) | P1v3 (Linux) |
| SQL Database | Basic (5 DTU) | S1 (20 DTU) |
| Key Vault | Standard | Standard |