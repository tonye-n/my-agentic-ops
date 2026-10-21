# Azure Agentic Operations

This repository provides a VS Code custom-agent workflow for turning an Azure
workload description into an architecture assessment, Bicep infrastructure, and
an `azd` provisioning attempt. The workflow separates architecture decisions
from infrastructure generation and deployment, and applies shared Azure naming,
SKU, and security defaults.

The included scenario describes a development web application hosted on Linux
App Service with Azure SQL Database and Key Vault. It contains infrastructure
and deployment artifacts, not the web application's source code; provisioning
does not publish an application.

## How the workflow works

Start the **01-conductor** agent in VS Code and provide the workload scenario.
It coordinates two specialized agents in order:

1. **02-architect** parses the request, assesses the Azure architecture, and
   writes `scenario/requirements-architecture.md`.
2. **03-bicep-deploy** reads that assessment and the shared defaults, creates
   the Bicep template, parameter file, and `azd` manifest, then attempts
   provisioning and records the result in `scenario/deployment-summary.md`.

The conductor checks the architecture artifact before continuing and checks
that the expected deployment artifacts exist before reporting the outcome. The
shared conventions and security baseline are in
[`SKILL.md`](.github/skills/SKILL.md); Bicep-specific standards are in
[`bicep.instructions.md`](.github/instructions/bicep.instructions.md).

## Included scenario

The current architecture targets **East US** (`eastus`) and includes:

- A Linux App Service plan (B1) and a Linux App Service configured for .NET 8.
- Azure SQL logical server and Basic (5 DTU) database.
- A Standard Key Vault configured for RBAC.
- A system-assigned App Service identity, with a Key Vault Secrets User role
  assignment.
- SQL Entra administrator configuration and a follow-up SQL script that grants
  the App Service identity reader and writer access to the database.

The App Service is HTTPS-only, uses a minimum TLS version of 1.2, and has
FTPS disabled. The template exports the App Service hostname, Key Vault URI,
and App Service identity principal ID.

The architecture assessment is
[`scenario/requirements-architecture.md`](scenario/requirements-architecture.md).
The Bicep template and parameters are in [`scenario/infra/`](scenario/infra/),
and [`scenario/azure.yaml`](scenario/azure.yaml) configures `azd` to use that
infrastructure directory.

## Prerequisites

- VS Code with custom agents and agent skills enabled.
- Azure CLI authenticated to the intended subscription (`az login`).
- Azure Developer CLI (`azd`) available in the terminal.
- An Azure subscription with the required permissions, regional resource
  availability, and quota for the selected SKUs.
- Azure CLI signed in as the Entra administrator to configure for Azure SQL.

Keep credentials and local Azure Developer CLI environment state out of source
control. The scenario's `.azure/` directory is ignored by Git. Supply the SQL
password through the environment; do not put it in a checked-in file or print
it in logs.

## Provisioning

The conductor workflow can generate the artifacts and attempt provisioning.
For manual provisioning, the runtime setup example is
[`env-variables.txt`](env-variables.txt). It selects the existing `webapp-dev`
`azd` environment and exports the variables below. Its shell block is wrapped
in an HTML comment, so use the commands below or remove the comment markers
before running the example:

| Variable | Value supplied by the example |
|---|---|
| `AZURE_SUBSCRIPTION_ID` | Subscription ID from the active Azure CLI account. Confirm that `az login` selected the intended subscription first. |
| `AZURE_LOCATION` | Azure region for the deployment. Set it to `eastus` to match the explicit `location` value in `scenario/infra/main.bicepparam`. |
| `SQL_ENTRA_ADMIN_OBJECT_ID` | Object ID of the signed-in Azure CLI user, used as the SQL Entra administrator. |
| `SQL_ENTRA_ADMIN_LOGIN` | UPN of the signed-in Azure CLI user, used as the SQL Entra administrator login. |
| `SQL_ADMIN_PASSWORD` | SQL administrator password entered at a hidden interactive prompt and exported only for the current shell process. |

Run the following from the repository root after `az login`. Use the intended
Azure SQL Entra administrator account. The password prompt does not echo the
password; enter a strong password that meets Azure SQL requirements.

```sh
cd scenario
azd env select webapp-dev

export AZURE_SUBSCRIPTION_ID="$(az account show --query id -o tsv)"
export AZURE_LOCATION=eastus
export SQL_ENTRA_ADMIN_OBJECT_ID="$(az ad signed-in-user show --query id -o tsv)"
export SQL_ENTRA_ADMIN_LOGIN="$(az ad signed-in-user show --query userPrincipalName -o tsv)"

read -r -s -p "Enter a new SQL administrator password (input hidden): " SQL_ADMIN_PASSWORD
printf '\n'
export SQL_ADMIN_PASSWORD

azd provision --no-prompt
```

The environment variables must remain exported in the same shell that runs
`azd provision`. If the `webapp-dev` `azd` environment does not exist yet, run
`azd env new webapp-dev --no-prompt` instead of `azd env select webapp-dev`.
Confirm the selected subscription, region, and SQL Entra administrator before
provisioning. Verify that the selected subscription has the required SKU
availability and quota in East US before deploying.

### Current deployment status

The artifacts in this checkout report that provisioning was **not attempted**.
The last preflight was for the previous UK South target: it found SQL
administrator inputs missing and a B1 App Service quota of zero. No new Azure
resources were created. That regional quota result does not establish capacity
in East US; supply the required inputs and verify the East US quota before
retrying. The deployment summary is
[`scenario/deployment-summary.md`](scenario/deployment-summary.md).

## After a successful deployment

The Bicep template creates the App Service managed identity and grants it
Key Vault secret-reading access, but it does not create the SQL database user.
After deployment, connect to `sqldb-webapp-dev` as the configured SQL Entra
administrator and run
[`grant-app-identity.sql`](scenario/sql/grant-app-identity.sql), passing the
`appServiceIdentityPrincipalId` deployment output as the SQLCMD variable
`AppServiceIdentityObjectId`. This grants the application identity the
`db_datareader` and `db_datawriter` roles.

The current `azd` manifest has no application services (`services: {}`), so
application code, application settings, and secret population are separate
follow-up work. Use managed identity and Key Vault for service-to-service
access; do not store passwords or connection strings in application settings.

## Repository layout

```text
.github/
  agents/          Conductor, architect, and Bicep/deploy agent definitions
  instructions/    Bicep authoring standards
  skills/          Shared Azure naming, security, and SKU defaults
env-variables.txt  Local provisioning command example and runtime environment setup
scenario/
  infra/           Bicep template and parameter file
  sql/             Post-deployment SQL identity authorization
  requirements-architecture.md
  azure.yaml
  deployment-summary.md
```
