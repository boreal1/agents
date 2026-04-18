# Microsoft Cloud Adoption Framework (CAF) — the Terraform view

The [Microsoft Cloud Adoption Framework for Azure](https://learn.microsoft.com/azure/cloud-adoption-framework/) is Microsoft's prescriptive guidance for adopting Azure at enterprise scale. The **Ready** methodology — and specifically the **Azure Landing Zones (ALZ)** architecture — is where CAF touches Terraform most directly. This file covers the opinions you should default to and the modules that implement them.

The short version: **do not design a landing zone from scratch**. CAF already answers the design questions; your job is to compose the CAF‑aligned AVM modules and pick the parameters. If a user is describing an enterprise Azure tenant, reach for this stack, not a pile of hand‑rolled `azurerm_*` resources.

## The CAF mental model

### Management‑group hierarchy
CAF prescribes a standard management‑group tree. The names below are conventions; the shape matters more than the exact names.

```text path=null start=null
Tenant Root
└── <intermediate-root>            # e.g. "alz" or your org code
    ├── Platform                   # platform team
    │   ├── Identity               # identity/AAD DS
    │   ├── Management             # LAW, Automation, Sentinel
    │   └── Connectivity           # hub networking, DNS, ExpressRoute
    ├── Landing zones              # workload-bearing subscriptions
    │   ├── Corp                   # internal, connected to the hub
    │   └── Online                 # internet-facing
    ├── Sandbox                    # short-lived experimentation
    └── Decommissioned             # subscriptions pending cancellation
```

Policy is assigned at these scopes, not at subscriptions or resource groups. That is the entire point of the hierarchy: one place to enforce baselines.

### Platform vs. application landing zones
- **Platform landing zones** host shared services consumed by workload teams: identity, management (logging/SIEM), connectivity (hub VNet, firewall, DNS). Owned by the platform team.
- **Application (workload) landing zones** host a single workload or bounded context. Owned by the workload team. Must conform to the guardrails set by the platform (policy, RBAC, networking peering to the hub).

Keep their Terraform codebases and state files separate. A platform change should not require a workload team to re‑plan, and vice versa.

### Subscription vending
A "vending machine" that creates a new subscription (or accepts an existing one), places it under the right management group, assigns RBAC and budgets, and optionally creates a seed resource group with VNet peering to the hub. This is the on‑ramp for application landing zones.

### Naming and tagging
CAF publishes recommended conventions — both for resource names and for required tags. Encoding them in modules rather than documentation is what actually makes them stick.

## The CAF‑aligned AVM stack

Use these together. Confirm the latest versions via `get_latest_module_version` before you pin — CAF modules move quickly and version numbers below are illustrative.

| Concern | Module | Purpose |
|---|---|---|
| Management groups + policy | `Azure/avm-ptn-alz/azurerm` | Deploys the CAF/ALZ management‑group hierarchy, policy definitions, policy set definitions, and policy assignments at the right scopes. The "brain" of the landing zone. |
| Management platform | `Azure/avm-ptn-alz-management/azurerm` | Deploys the central Log Analytics workspace, Automation account, Sentinel, and platform monitoring. Consumed by every subscription via diagnostic settings. |
| Hub networking | `Azure/avm-ptn-hubnetworking/azurerm` | Hub‑spoke connectivity: hub VNet, Azure Firewall, VPN/ExpressRoute gateways, Private DNS zones. |
| Subscription vending | `Azure/avm-ptn-alz-sub-vending/azure` | Creates/imports a subscription, places it under a management group, assigns RBAC/budgets, optionally seeds resource groups, UAMIs (with federated credentials), network security groups, route tables, and spoke VNets with hub peering or Virtual WAN connections. Supersedes the now‑deprecated `Azure/lz-vending/azurerm`. **Note the provider slug is `azure`, not `azurerm`** — this is an `azapi`‑based module and the registry address reflects that. |
| Naming | `Azure/naming/azurerm` | Returns CAF‑compliant names for every resource type (`name`, `name_unique`, `slug`). |
| Regions | `Azure/avm-utl-regions/azurerm` | Region metadata: which regions are paired, which support zones, which are AZ‑enabled. |

Legacy notes:

- `Azure/caf-enterprise-scale/azurerm` (the "CAF Enterprise‑Scale" module) was the pre‑AVM way to stand up ALZ on Terraform. For new deployments prefer the AVM stack above. Keep using `caf-enterprise-scale` only when maintaining an existing state that already manages the hierarchy through it — migrating is a state surgery project, not a one‑line change.
- `Azure/lz-vending/azurerm` is the deprecated predecessor to `Azure/avm-ptn-alz-sub-vending/azure`. The replacement is the AVM‑branded, `azapi`‑native module with the same mental model (subscription alias → management group association → optional seeded resources) but a refreshed input surface and lifecycle. Use the AVM module for all new vending; migrate existing `lz-vending` state only when you have a reason to touch it (breaking change, variable rename, etc.). The original `lz-vending` repo has been rebranded to the AVM module; do not pin new work to the deprecated source.

## Typical topologies

### Greenfield: one tenant, full ALZ
Four separate state files, four separate pipelines, four separate subscriptions (or at least clear RBAC boundaries):

1. `platform/alz-management-groups` — calls `avm-ptn-alz`. Lives at the tenant root; deployed by a privileged identity.
2. `platform/management` — calls `avm-ptn-alz-management` in the **Management** subscription.
3. `platform/connectivity` — calls `avm-ptn-hubnetworking` in the **Connectivity** subscription.
4. `platform/vending` — calls `Azure/avm-ptn-alz-sub-vending/azure` (one invocation per new workload subscription) in a platform‑owned subscription. Each invocation creates (or imports) the subscription, associates it with the target management group, and optionally lays down an initial resource group, UAMI with federated credentials for CI/CD, NSGs, and a spoke VNet peered to the hub.

Each workload team then gets their own repo(s) that consume outputs from the platform states (hub VNet ID, shared LAW ID, DNS zone IDs) via data sources or `terraform_remote_state`.

### Brownfield: existing tenant
Two realistic paths:

1. **Import** the existing management‑group hierarchy into `avm-ptn-alz` using `import {}` blocks (Terraform ≥ 1.5) and converge. Expensive but produces a clean end state.
2. **Wrap** — leave the hierarchy alone and adopt only the bits you need (management platform, hub networking, vending). Accept that policy management stays wherever it is.

Pick explicitly; do not drift into a third hybrid state.

## Naming with `Azure/naming/azurerm`

Stop hand‑writing names. Instead:

```hcl path=null start=null
module "naming" {
  source  = "Azure/naming/azurerm"
  version = "~> 0.4"   # confirm via get_latest_module_version

  prefix = ["contoso", "corp", "prod"]   # or use suffix depending on the scheme
}

module "resource_group" {
  source  = "Azure/avm-res-resources-resourcegroup/azurerm"
  version = "~> 0.2"

  name     = module.naming.resource_group.name_unique
  location = var.location
}

module "storage_account" {
  source  = "Azure/avm-res-storage-storageaccount/azurerm"
  version = "~> 0.6"

  name                = module.naming.storage_account.name_unique
  location            = var.location
  resource_group_name = module.resource_group.name
}
```

Benefits:

- Length/charset rules are enforced per resource type (storage accounts are lowercase, globally unique, 3‑24 chars — the module handles it).
- `name_unique` adds a random suffix where global uniqueness is required.
- Swapping `prefix` or `suffix` once changes every name everywhere consistently.

## Tagging

CAF expects a minimum set of tags on every resource. A pragmatic baseline you can codify:

```hcl path=null start=null
variable "tags" {
  type = object({
    environment      = string   # e.g. "prod", "nonprod", "sandbox"
    workload         = string   # e.g. "orders-api"
    business_unit    = string
    cost_center      = string
    data_classification = string   # "public" | "internal" | "confidential" | "restricted"
    owner_email      = string   # team DL, not a person
  })
  description = "CAF-baseline tags applied to every resource."

  validation {
    condition     = contains(["public", "internal", "confidential", "restricted"], var.tags.data_classification)
    error_message = "data_classification must be one of: public, internal, confidential, restricted."
  }
}

locals {
  caf_tags = merge(var.tags, {
    managed_by = "terraform"
  })
}
```

Then pass `local.caf_tags` to every AVM module's `tags` input. Enforce the schema with Azure Policy (`Require a tag`, `Append tag`, `Inherit a tag`) at the **Landing zones** management group — `avm-ptn-alz` ships definitions for these.

## RBAC and identity

- **Every** data‑plane access goes through Azure RBAC, not legacy access policies. The AVM key vault, storage account, and similar modules expose `role_assignments` maps — use them.
- Prefer **Managed Identities** (system‑assigned where possible, user‑assigned when resources share an identity) over service principals with secrets.
- For CI/CD use **OIDC federation** (GitHub Actions / Azure DevOps workload identity federation) against a deployment SPN/UAMI scoped to the specific management group the pipeline is allowed to touch. `avm-ptn-alz-sub-vending`'s `user_managed_identities` input can provision the UAMI **and** its GitHub / Azure DevOps / Terraform Cloud federated credentials in the same plan as the subscription — prefer that over creating them in a separate step.

## Policy

Azure Policy is the enforcement layer CAF relies on. `avm-ptn-alz` assigns the ALZ policy set (ALZ‑managed initiatives) at the appropriate scopes. Workload teams should not assign tenant‑wide policy; they may assign workload‑scoped exemptions or additional policies at their subscription scope with platform‑team approval.

When reviewing CAF Terraform, check that:

- Tenant‑root / intermediate‑root scopes have ALZ baseline assigned.
- Landing‑zone groups have `Deny public endpoints`, `Require private link`, diagnostic‑settings policies assigned (via `avm-ptn-alz`'s `policy_assignments_to_modify` input or the defaults).
- Platform policy changes flow through a separate pipeline from workload changes.

## A minimal reference layout

```text path=null start=null
acme-landing-zone/
├── platform/
│   ├── management-groups/     # avm-ptn-alz
│   │   ├── terraform.tf
│   │   ├── providers.tf       # provider aliases for each subscription
│   │   ├── backend.tf
│   │   └── main.tf
│   ├── management/            # avm-ptn-alz-management (in Management sub)
│   ├── connectivity/          # avm-ptn-hubnetworking (in Connectivity sub)
│   └── vending/               # avm-ptn-alz-sub-vending (one file per workload subscription)
├── workloads/
│   ├── orders-api/
│   │   ├── dev/
│   │   ├── prod/
│   │   └── modules/
│   └── payments-api/
└── shared/
    └── tags.tf                # shared tag schema/validation referenced by workloads
```

Each directory is its own Terraform root with its own state, backend, and pipeline.

## Review checklist (use during code review)

- Is the management‑group hierarchy managed by `avm-ptn-alz` (greenfield) or explicitly documented as maintained by `caf-enterprise-scale`?
- Is subscription vending using `avm-ptn-alz-sub-vending` (not the deprecated `lz-vending`)?
- Does every resource name come from `Azure/naming/azurerm` (or an equivalent typed naming module)?
- Are the CAF baseline tags present and validated?
- Are platform and workload states separated?
- Does the workload subscription consume shared resources (hub VNet, LAW) via data sources or `terraform_remote_state`, not by duplicating them?
- Is policy assigned at management‑group scope, not per resource group?
- Is data‑plane access granted via Azure RBAC (not legacy access policies, not storage account keys)?
- Is CI/CD using OIDC federation with a deployment identity scoped to a specific management group?

If any answer is "no", raise it in review before `apply`.
