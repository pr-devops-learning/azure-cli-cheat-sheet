# Azure CLI Cheat Sheet (`az`)
### Practical command reference for Azure Administrators and Cloud Engineers

> **Scope:** This is a practical, expandable cheat sheet for commonly used Azure CLI command groups—not a literal listing of every CLI command. Microsoft documents thousands of commands, and syntax may vary by CLI version, resource provider, cloud, and installed extensions. Verify unfamiliar commands with `az <group> <command> --help` and the official reference: https://learn.microsoft.com/en-us/cli/azure/

## Contents
- [1. Command pattern and help](#1-command-pattern-and-help)
- [2. Install and verify](#2-install-and-verify)
- [3. Sign in, tenants, subscriptions](#3-sign-in-tenants-subscriptions)
- [4. Output, filtering, defaults](#4-output-filtering-defaults)
- [5. Resource groups and resources](#5-resource-groups-and-resources)
- [6. Tags, locks, providers](#6-tags-locks-providers)
- [7. Virtual machines, disks, snapshots](#7-virtual-machines-disks-snapshots)
- [8. Networking](#8-networking)
- [9. Storage](#9-storage)
- [10. Identity, RBAC and managed identities](#10-identity-rbac-and-managed-identities)
- [11. Key Vault](#11-key-vault)
- [12. App Service, Functions and containers](#12-app-service-functions-and-containers)
- [13. AKS](#13-aks)
- [14. Monitoring and troubleshooting](#14-monitoring-and-troubleshooting)
- [15. Backup and recovery discovery](#15-backup-and-recovery-discovery)
- [16. Policy, security and cost discovery](#16-policy-security-and-cost-discovery)
- [17. Database examples](#17-database-examples)
- [18. Deployments and Infrastructure as Code](#18-deployments-and-infrastructure-as-code)
- [19. Useful scripting patterns](#19-useful-scripting-patterns)
- [20. Safety checklist](#20-safety-checklist)
- [21. Official references](#21-official-references)

## 1. Command pattern and help

General pattern:

```bash
az <command-group> <subgroup> <action> [options]
```

Examples:
- `az vm list` — list virtual machines.
- `az network vnet show ...` — inspect a virtual network.
- `az storage account create ...` — create a storage account.

| Command | What it does |
|---|---|
| `az --help` | Lists top-level commands and help |
| `az find "az vm"` | Searches for common examples |
| `az vm --help` | Shows commands and options in a group |
| `az vm create --help` | Shows details for one command |
| `az version` | Shows installed CLI version and dependencies |
| `az upgrade` | Upgrades Azure CLI where supported |
| `az extension list -o table` | Lists installed extensions |
| `az extension add --name <extension>` | Installs an extension (verify its publisher and trust first) |

## 2. Install and verify

Install instructions for Windows, Linux, macOS, Cloud Shell and containers: https://learn.microsoft.com/en-us/cli/azure/install-azure-cli

```bash
az version
az --help
az account show -o table
```

Azure Cloud Shell is a browser-based option with Azure CLI available: https://learn.microsoft.com/en-us/azure/cloud-shell/overview

## 3. Sign in, tenants, subscriptions

| Command | Explanation / example |
|---|---|
| `az login` | Interactive sign-in |
| `az login --tenant <tenant-id>` | Sign in to a specific tenant |
| `az account list -o table` | List subscriptions visible to you |
| `az account show -o jsonc` | Show active subscription and tenant |
| `az account set --subscription "<subscription-id-or-name>"` | Select the subscription for subsequent commands |
| `az account list --all -o table` | List subscriptions available across accessible tenants, where supported |
| `az logout` | Sign out of Azure CLI |
| `az config get` | Inspect CLI configuration |
| `az configure --defaults group=<rg> location=<region>` | Set convenient defaults; confirm the active context before using them |

**Safety habit:** Run `az account show -o table` before making changes. A subscription name can be similar to another environment's name.

## 4. Output, filtering, defaults

Azure CLI output formats include `json`, `jsonc`, `table`, `tsv`, `yaml`, `yamlc`, and `none`.

```bash
az group list -o table
az vm list -o jsonc
az vm list --query "[].{Name:name,RG:resourceGroup,Location:location}" -o table
az vm list --query "[?powerState=='VM running'].{Name:name,RG:resourceGroup}" -o table
az account show --query "{Subscription:name,Id:id,Tenant:tenantId}" -o json
```

- `-o table` / `--output table`: readable tables.
- `-o tsv`: convenient for shell scripting.
- `--query`: JMESPath filter or projection; query fields vary by command.
- `--debug`: verbose diagnostic output; may contain sensitive operational details, so review before sharing logs.
- `--only-show-errors`: suppresses warnings and informational output, not a fix for errors.

## 5. Resource groups and resources

Set variables first (Bash examples):

```bash
RG="rg-demo-eastus"
LOC="eastus"
SUB="<subscription-id>"
az account set --subscription "$SUB"
```

| Command | Explanation / example |
|---|---|
| `az group list -o table` | List resource groups |
| `az group show -n "$RG" -o jsonc` | Inspect a resource group |
| `az group exists -n "$RG"` | Check whether a group exists |
| `az group create -n "$RG" -l "$LOC"` | Create a resource group |
| `az resource list -g "$RG" -o table` | List resources in a group |
| `az resource show --ids <resource-id>` | Inspect a resource by full resource ID |
| `az resource tag --ids <resource-id> --tags Environment=Dev Owner=Platform` | Apply tags to a resource; check tag operation semantics first |
| `az group delete -n "$RG" --no-wait` | **Destructive:** deletes the group and its resources; use only after review |

Resource group example:

```bash
az group create --name rg-learning-eastus --location eastus
az group list --query "[].{Name:name,Location:location}" -o table
```

## 6. Tags, locks, providers

```bash
# Add/update tags on a resource group
az group update -n "$RG" --set tags.Environment=Dev tags.Owner=CloudOps

# List resource locks in a resource group
az lock list --resource-group "$RG" -o table

# Create a delete-protection lock (test its operational impact)
az lock create --name DoNotDelete --lock-type CanNotDelete --resource-group "$RG"

# List registered resource providers
az provider list --query "[?registrationState=='NotRegistered'].namespace" -o tsv

# Register a provider only when needed and authorized
az provider register --namespace Microsoft.Compute
```

Locks are not a substitute for backups or access controls. Some operations may be blocked while a lock is present.

## 7. Virtual machines, disks, snapshots

```bash
# List VMs
az vm list -d -o table

# Inspect a VM
az vm show -g "$RG" -n <vm-name> -d -o jsonc

# Power state
az vm get-instance-view -g "$RG" -n <vm-name> --query "instanceView.statuses[].displayStatus" -o table

# Start, stop/deallocate, restart
az vm start -g "$RG" -n <vm-name>
az vm deallocate -g "$RG" -n <vm-name>
az vm restart -g "$RG" -n <vm-name>

# Resize: first check supported sizes and current constraints
az vm list-vm-resize-options -g "$RG" -n <vm-name> -o table
az vm resize -g "$RG" -n <vm-name> --size <vm-size>

# List disks / snapshots
az disk list -g "$RG" -o table
az snapshot list -g "$RG" -o table

# Run a command on a VM (permissions and VM agent required)
az vm run-command invoke -g "$RG" -n <vm-name> --command-id RunPowerShellScript --scripts "Get-Date"
```

Create a VM only after deciding OS image, authentication, network exposure, disk, SKU, availability, and cost:

```bash
az vm create \
  --resource-group "$RG" \
  --name vm-demo-01 \
  --image Ubuntu2204 \
  --size Standard_B2s \
  --admin-username azureuser \
  --generate-ssh-keys
```

This example is illustrative; availability and pricing vary by region and subscription. Do not expose SSH/RDP to the entire internet unless there is a justified, controlled requirement. Prefer Bastion, VPN, private access, or tightly restricted source IPs.

## 8. Networking

### Virtual networks and subnets

```bash
az network vnet list -o table
az network vnet show -g "$RG" -n <vnet-name> -o jsonc
az network vnet subnet list -g "$RG" --vnet-name <vnet-name> -o table

az network vnet create \
  -g "$RG" -n vnet-demo \
  --address-prefixes 10.10.0.0/16 \
  --subnet-name snet-app \
  --subnet-prefixes 10.10.1.0/24
```

### NSGs and rules

```bash
az network nsg list -o table
az network nsg show -g "$RG" -n <nsg-name> -o jsonc
az network nsg rule list -g "$RG" --nsg-name <nsg-name> -o table
```

Create a rule only after checking direction, priority, source/destination, protocol, ports and business need:

```bash
az network nsg rule create \
  -g "$RG" --nsg-name <nsg-name> -n Allow-HTTPS-From-ApprovedRange \
  --priority 200 --direction Inbound --access Allow --protocol Tcp \
  --source-address-prefixes <approved-cidr> \
  --source-port-ranges '*' \
  --destination-address-prefixes '*' \
  --destination-port-ranges 443
```

### Public IP, NIC, routes, DNS and load balancers

```bash
az network public-ip list -o table
az network nic list -o table
az network route-table list -o table
az network dns zone list -o table
az network lb list -o table
az network application-gateway list -o table
az network vnet-gateway list -o table
az network watcher list -o table
```

Inspect a specific resource using `show`, and use the relevant subgroup's `--help` before creating or changing network resources. VPN, Firewall, Private Link, Front Door, DNS, and load balancer configurations have service-specific prerequisites and potential cost impact.

## 9. Storage

```bash
# List and inspect accounts
az storage account list -o table
az storage account show -g "$RG" -n <storage-account> -o jsonc

# Create a storage account (name must be globally unique; SKU and redundancy affect cost)
az storage account create \
  -g "$RG" -n <globally-unique-name> -l "$LOC" \
  --sku Standard_LRS --kind StorageV2 \
  --min-tls-version TLS1_2

# Blob containers
az storage container list --account-name <storage-account> --auth-mode login -o table
az storage container create --account-name <storage-account> --name <container> --auth-mode login
az storage blob list --account-name <storage-account> --container-name <container> --auth-mode login -o table

# File shares
az storage share-rm list --resource-group "$RG" --storage-account <storage-account> -o table
```

Prefer Microsoft Entra ID authentication (`--auth-mode login`) and least-privilege data roles where supported. Avoid putting account keys or SAS tokens in source control, command history, screenshots or public issue reports.

## 10. Identity, RBAC and managed identities

```bash
# Show signed-in identity context
az account show -o jsonc

# List role definitions and assignments
az role definition list --name Reader -o jsonc
az role assignment list --assignee <user-or-object-id> --all -o table
az role assignment list --scope <resource-id> -o table

# Assign least-privilege role at the narrowest practical scope
az role assignment create \
  --assignee <user-or-service-principal-object-id> \
  --role Reader \
  --scope <resource-id>

# VM managed identity
az vm identity show -g "$RG" -n <vm-name>
az vm identity assign -g "$RG" -n <vm-name>
```

RBAC changes are security-sensitive. Verify the assignee object ID, role, scope, tenant and subscription. Avoid broad `Owner` or `Contributor` assignments unless genuinely required. Entra ID CLI commands under `az ad` may depend on Microsoft Graph permissions and current CLI support.

## 11. Key Vault

```bash
az keyvault list -o table
az keyvault show -g "$RG" -n <vault-name> -o jsonc
az keyvault secret list --vault-name <vault-name> -o table
az keyvault secret show --vault-name <vault-name> --name <secret-name>
az keyvault key list --vault-name <vault-name> -o table
az keyvault certificate list --vault-name <vault-name> -o table
```

Secret retrieval may expose sensitive values in terminal output. Do not paste retrieved values into a public repository or logs. Vault authorization mode, firewall/private endpoint rules and RBAC permissions must be configured correctly. Prefer managed identity rather than embedding credentials in scripts.

## 12. App Service, Functions and containers

```bash
# App Service plans and web apps
az appservice plan list -o table
az webapp list -o table
az webapp show -g "$RG" -n <app-name> -o jsonc
az webapp restart -g "$RG" -n <app-name>
az webapp log tail -g "$RG" -n <app-name>

# Function Apps
az functionapp list -o table
az functionapp show -g "$RG" -n <function-app> -o jsonc

# Container Registry
az acr list -o table
az acr show -g "$RG" -n <registry-name> -o jsonc
az acr login --name <registry-name>
az acr repository list --name <registry-name> -o table

# Azure Container Apps
az containerapp list -g "$RG" -o table
az containerapp show -g "$RG" -n <container-app> -o jsonc
```

Deployment commands vary by runtime, source type, registry and hosting plan. Review authentication, networking, secrets, health checks, deployment slots, and cost before deploying.

## 13. AKS

```bash
az aks list -o table
az aks show -g "$RG" -n <aks-name> -o jsonc
az aks get-credentials -g "$RG" -n <aks-name>
az aks nodepool list -g "$RG" --cluster-name <aks-name> -o table
az aks nodepool show -g "$RG" --cluster-name <aks-name> -n <pool-name> -o jsonc
```

`az aks get-credentials` modifies local kubeconfig. Review the target cluster/context before running `kubectl` commands. AKS creation, upgrades and scaling can be costly and affect availability.

## 14. Monitoring and troubleshooting

```bash
# Activity logs
az monitor activity-log list --resource-group "$RG" --offset 1d -o table
az monitor activity-log list --status Failed --offset 6h -o jsonc

# Metrics definitions and metrics
az monitor metrics list-definitions --resource <resource-id> -o table
az monitor metrics list --resource <resource-id> --metric <metric-name> --interval PT5M -o jsonc

# Alerts and diagnostic settings
az monitor alert list -g "$RG" -o table
az monitor diagnostic-settings list --resource <resource-id> -o jsonc

# Resource health and support discovery
az resource show --ids <resource-id> -o jsonc
az monitor --help
az support --help
```

Log Analytics query command availability and syntax may depend on installed extensions and workspace configuration. Discover the exact subgroup with `az monitor log-analytics --help`. Time ranges and metric names vary by service.

## 15. Backup and recovery discovery

Azure Backup and Site Recovery command groups and capabilities vary by workload and recovery configuration. Start by exploring the supported commands:

```bash
az backup --help
az backup vault list -o table
az backup protection --help
az backup recoverypoint --help
az backup restore --help
az site-recovery --help
```

Before changing protection or restoring data, verify vault, policy, protected item, recovery point, target region and restore destination. Test recovery procedures periodically; a command returning success is not a complete recovery validation.

## 16. Policy, security and cost discovery

```bash
# Azure Policy
az policy definition list -o table
az policy assignment list --scope <subscription-or-resource-group-scope> -o table
az policy state list --resource-group "$RG" -o table

# Security posture / Defender for Cloud command discovery
az security --help

# Advisor recommendations
az advisor recommendation list -o table

# Cost-management commands and current supported syntax
az costmanagement --help
az consumption --help
```

Some cost-management operations require specific billing agreement types, permissions, scopes or API versions. Use the official subgroup help and verify the date range, currency and scope before relying on totals.

## 17. Database examples

```bash
# Azure SQL
az sql server list -o table
az sql db list -g "$RG" -s <sql-server> -o table

# PostgreSQL / MySQL flexible server command discovery
az postgres flexible-server list -o table
az mysql flexible-server list -o table

# Cosmos DB
az cosmosdb list -o table
az cosmosdb show -g "$RG" -n <cosmos-account> -o jsonc
```

Database names, SKUs, networking, backup retention, authentication and regional availability are service-specific. Do not assume the sample command creates a production-ready database.

## 18. Deployments and Infrastructure as Code

```bash
# Resource group deployments
az deployment group list -g "$RG" -o table
az deployment group show -g "$RG" -n <deployment-name> -o jsonc
az deployment group validate -g "$RG" --template-file main.bicep
az deployment group what-if -g "$RG" --template-file main.bicep
az deployment group create -g "$RG" -f main.bicep

# Subscription-scope deployments
az deployment sub what-if --location "$LOC" --template-file main.bicep
az deployment sub create --location "$LOC" --template-file main.bicep

# Bicep tooling
az bicep version
az bicep build --file main.bicep

# Terraform is a separate CLI; Azure CLI can authenticate to Azure for workflows
terraform init
terraform fmt -check
terraform validate
terraform plan
terraform apply
```

**Terraform does not replace Azure CLI.** Terraform is typically used to define and manage infrastructure declaratively; Azure CLI is useful for authentication, discovery, diagnostics and one-off operational tasks. Use one clear owner for each resource to avoid Terraform state drift or conflicting changes.

## 19. Useful scripting patterns

### Bash: loop over resource groups

```bash
for rg in $(az group list --query "[].name" -o tsv); do
  echo "Resource group: $rg"
  az resource list --resource-group "$rg" --query "[].{Name:name,Type:type,Location:location}" -o table
done
```

### PowerShell: capture a resource list

```powershell
$resourceGroup = "rg-demo-eastus"
$resources = az resource list --resource-group $resourceGroup | ConvertFrom-Json
$resources | Select-Object name, type, location | Format-Table
```

### Save command output for review

```bash
az vm list -d -o json > vm-inventory.json
```

Treat exported inventory as potentially sensitive: it can reveal internal names, topology and operational details.

## 20. Safety checklist

Before running a command that changes Azure:
1. Confirm `az account show -o table` points to the intended tenant and subscription.
2. Confirm resource group, resource name, region and scope.
3. Review `--help`, required permissions, service prerequisites and expected cost.
4. Use read-only `list` / `show` first; use `what-if` where supported.
5. Check whether the command starts billable capacity, changes security exposure or affects production availability.
6. Back up data and have a recovery plan before destructive changes.
7. Never publish secrets, access tokens, private keys, SAS tokens, connection strings, subscription-specific inventories or sensitive logs.
8. Prefer managed identities and least-privilege RBAC for automation.
9. Test in a non-production subscription/resource group first.
10. Review CLI exit status and service state after execution.

**High-risk actions to review carefully:** `delete`, `purge`, `role assignment create/delete`, firewall/NSG changes, key regeneration, storage access changes, VM resizing/deallocation, database changes, backup deletion and restore operations.

## 21. Official references

- Azure CLI home and installation: https://learn.microsoft.com/en-us/cli/azure/
- Complete A–Z reference index: https://learn.microsoft.com/en-us/cli/azure/reference-index
- Command reference index of documented groups: https://learn.microsoft.com/en-us/cli/azure/reference-docs-index
- Azure CLI samples: https://github.com/Azure/azure-cli-samples
- Microsoft Azure CLI documentation source: https://github.com/MicrosoftDocs/azure-docs-cli
- GitHub Pages setup: https://docs.github.com/en/pages/getting-started-with-github-pages/creating-a-github-pages-site

---

## Contributing

If you find a command that has changed:
1. Confirm it with `az <group> <command> --help`.
2. Check the Microsoft Learn command reference.
3. Test in a non-production environment.
4. Update the example with placeholders rather than real tenant, subscription, account or secret values.
5. Submit a pull request with the CLI version and the test context (without sensitive data).
