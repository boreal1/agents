# Azure Verified Modules (AVM)

AVM is Microsoft's curated set of Terraform (and Bicep) modules that implement the [AVM specifications](https://azure.github.io/Azure-Verified-Modules/). When writing Terraform for Azure, the default choice should be an AVM module — not a hand‑rolled `azurerm_*` block, and not a legacy community module.

## Naming and what each tier is for
AVM uses a strict naming scheme you can rely on when searching the registry:

- `Azure/avm-res-<resource-provider>-<resource>/azurerm` — **Resource modules**. One Azure resource (plus its natural ancillaries like diagnostics, RBAC, private endpoints, CMK). These are your building blocks.
  - Examples: `Azure/avm-res-resources-resourcegroup/azurerm`, `Azure/avm-res-storage-storageaccount/azurerm`, `Azure/avm-res-network-virtualnetwork/azurerm`, `Azure/avm-res-keyvault-vault/azurerm`, `Azure/avm-res-compute-virtualmachine/azurerm`, `Azure/avm-res-containerservice-managedcluster/azurerm`.
- `Azure/avm-ptn-<pattern>/azurerm` — **Pattern modules**. Opinionated compositions of resource modules for a named scenario (hub‑spoke networking, AVD, landing zones, ALZ management groups, etc.). Use them when they match your scenario; otherwise compose resource modules yourself.
- `Azure/avm-utl-<name>/azurerm` — **Utility modules**. Helpers that return values, not resources (naming, regions, interfaces).

Always confirm the specific module exists and get its latest version via `get_latest_module_version`. Then use `search_modules` → `get_module_details` to read its full input/output contract.

## The standard AVM interface
Every AVM resource module exposes the same cross‑cutting inputs. Learning this interface once lets you wire any AVM module without re‑reading the whole README:

- `name`, `location`, `resource_group_name` (or equivalent scope).
- `tags` — module merges these with any resource‑specific tagging logic.
- `enable_telemetry` — defaults to `true`; sends anonymous usage data to Microsoft via a no‑op deployment. Set `false` if your org forbids it.
- `lock` — object `{ kind = "CanNotDelete" | "ReadOnly", name = string }` to apply a management lock.
- `role_assignments` — map of RBAC assignments; scoped to the resource.
- `diagnostic_settings` — map of diagnostic settings (destinations: Log Analytics, Event Hub, Storage).
- `private_endpoints` — map of private endpoints with DNS zone group wiring.
- `managed_identities` — `{ system_assigned = bool, user_assigned_resource_ids = set(string) }`.
- `customer_managed_key` — object enabling CMK (key vault key reference + UAMI).

Every AVM module exposes at least: `resource` (the full resource object), `resource_id`, `name`, and a set of related IDs (private endpoint IDs, private endpoint application security group associations, etc.).

## Selection matrix
- Single Azure resource, no opinionated composition needed → **resource module**.
- Multi‑resource scenario that matches a named pattern (hub‑spoke, AVD host pool + session hosts, ALZ) → **pattern module**.
- Enterprise landing zone / management‑group hierarchy / platform subscriptions → CAF‑aligned pattern stack (`avm-ptn-alz`, `avm-ptn-alz-management`, `avm-ptn-hubnetworking`, and `avm-ptn-alz-sub-vending` for subscription vending — which supersedes the deprecated `Azure/lz-vending/azurerm`). See `references/caf.md` for the full workflow — do not reinvent this from `azurerm_management_group` / `azurerm_policy_definition` blocks.
- Need a naming convention, region mapping, or interface type you'd otherwise duplicate → **utility module**. For CAF‑compliant names specifically, use `Azure/naming/azurerm`.
- Nothing fits and the resource is stable in `azurerm` → drop to `azurerm_*` directly, but wrap it in your own small module if it will be reused.

## Worked example: a small stack
Resource group + virtual network + storage account with diagnostics to Log Analytics. All versions should be re‑confirmed via MCP at the time of writing — the pins below are illustrative.

```hcl path=null start=null
# terraform.tf
terraform {
  required_version = ">= 1.9.0"

  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 4.69"    # get_latest_provider_version: hashicorp/azurerm
    }
    azapi = {
      source  = "Azure/azapi"
      version = "~> 2.9"     # get_latest_provider_version: Azure/azapi
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.6"
    }
  }
}
```

```hcl path=null start=null
# providers.tf
provider "azurerm" {
  features {}
}

provider "azapi" {}
```

```hcl path=null start=null
# variables.tf
variable "location" {
  type        = string
  description = "Azure region for all resources in this stack."
  default     = "eastus"
}

variable "name_prefix" {
  type        = string
  description = "Short lowercase prefix used to name resources (3-10 chars, letters/digits only)."
  validation {
    condition     = can(regex("^[a-z0-9]{3,10}$", var.name_prefix))
    error_message = "name_prefix must be 3-10 lowercase letters or digits."
  }
}

variable "tags" {
  type        = map(string)
  description = "Tags applied to every resource in the stack."
  default     = {}
}
```

```hcl path=null start=null
# main.tf
resource "random_string" "suffix" {
  length  = 6
  lower   = true
  upper   = false
  numeric = true
  special = false
}

module "resource_group" {
  source  = "Azure/avm-res-resources-resourcegroup/azurerm"
  version = "~> 0.2"    # get_latest_module_version

  name     = "rg-${var.name_prefix}-${random_string.suffix.result}"
  location = var.location
  tags     = var.tags
}

module "log_analytics" {
  source  = "Azure/avm-res-operationalinsights-workspace/azurerm"
  version = "~> 0.4"    # get_latest_module_version

  name                = "log-${var.name_prefix}"
  location            = var.location
  resource_group_name = module.resource_group.name
  tags                = var.tags
}

module "virtual_network" {
  source  = "Azure/avm-res-network-virtualnetwork/azurerm"
  version = "~> 0.17"   # get_latest_module_version

  name                = "vnet-${var.name_prefix}"
  location            = var.location
  resource_group_name = module.resource_group.name
  address_space       = ["10.20.0.0/16"]

  subnets = {
    app = {
      name             = "snet-app"
      address_prefixes = ["10.20.1.0/24"]
    }
    data = {
      name             = "snet-data"
      address_prefixes = ["10.20.2.0/24"]
    }
  }

  tags = var.tags
}

module "storage_account" {
  source  = "Azure/avm-res-storage-storageaccount/azurerm"
  version = "~> 0.6"    # get_latest_module_version

  name                     = "st${var.name_prefix}${random_string.suffix.result}"
  location                 = var.location
  resource_group_name      = module.resource_group.name
  account_tier             = "Standard"
  account_replication_type = "ZRS"

  # Standard AVM interface — works identically across resource modules.
  diagnostic_settings = {
    to_law = {
      name                  = "diag-to-law"
      workspace_resource_id = module.log_analytics.resource_id
      log_categories        = ["StorageRead", "StorageWrite", "StorageDelete"]
      metric_categories     = ["AllMetrics"]
    }
  }

  managed_identities = {
    system_assigned = true
  }

  tags = var.tags
}
```

```hcl path=null start=null
# outputs.tf
output "resource_group_name" {
  description = "Name of the resource group that holds the stack."
  value       = module.resource_group.name
}

output "storage_account_id" {
  description = "Resource ID of the storage account."
  value       = module.storage_account.resource_id
}

output "virtual_network_id" {
  description = "Resource ID of the virtual network."
  value       = module.virtual_network.resource_id
}
```

Finish with `terraform fmt -recursive && terraform init -backend=false && terraform validate` — all three must succeed before you claim the stack is ready.

## Tips when composing AVMs
- Do not re‑implement diagnostics, RBAC, private endpoints, or locks yourself — use the AVM's standard inputs. That's the whole point.
- AVM modules use Terraform `~> 1.9` features in some versions; check each module's `required_version`.
- Mix‑and‑match `azurerm_*` with AVMs is fine — pass the AVM output IDs to your raw resources.
- For Azure‑only resources that `azurerm` has not implemented yet, use the `azapi` provider (resource: `azapi_resource`) — AVM modules themselves lean on it for newer RPs.
