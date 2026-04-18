# State, backends, and refactoring

The state file is the most valuable artifact in a Terraform codebase — it is the map between your configuration and real cloud resources. Treat it carefully.

## Always use a remote backend
Local state (`terraform.tfstate` on disk) is fine for experimenting and nothing else. In any environment more than one human or pipeline touches, configure a remote backend.

### Azure (preferred — Azure Storage)
Native support for state locking landed in AzureRM long ago; there is no separate lock table required.

```hcl path=null start=null
terraform {
  backend "azurerm" {
    resource_group_name  = "rg-tfstate-prod"
    storage_account_name = "sttfstateprod"
    container_name       = "tfstate"
    key                  = "platform/network.tfstate"
    use_azuread_auth     = true
    subscription_id      = "00000000-0000-0000-0000-000000000000"
    tenant_id            = "00000000-0000-0000-0000-000000000000"
  }
}
```

Prefer `use_azuread_auth = true` with an identity that has the **Storage Blob Data Contributor** role on the container — it avoids storage account keys entirely.

### AWS
As of Terraform 1.10+, the `s3` backend supports native state locking via S3 conditional writes (`use_lockfile = true`), removing the need for a DynamoDB table. Use this for new backends. Retain DynamoDB only for older Terraform versions.

```hcl path=null start=null
terraform {
  backend "s3" {
    bucket       = "acme-tfstate-prod"
    key          = "platform/network.tfstate"
    region       = "us-east-1"
    encrypt      = true
    use_lockfile = true    # native S3 locking, Terraform ≥ 1.10
  }
}
```

### HCP Terraform / Terraform Enterprise
`cloud {}` block. Adds run history, policies, and the UI. Worth it for teams.

## Environment isolation
Two defensible options — pick one per codebase and stick with it.

1. **Separate backend keys per environment** (preferred). One directory per environment (`envs/dev`, `envs/prod`) with its own `backend.tf` pointing at a distinct key. Strong isolation, no cross‑env blast radius, different backends/credentials per env possible.
2. **Workspaces**. Same code, `terraform workspace new dev` / `prod`. Fine for quick experiments. Weaker isolation — same backend credentials, same state account — and the `terraform.workspace` interpolation quickly gets hacky.

Avoid mixing the two. Pick and document.

## Splitting state
Once a state has more than ~200 resources, operations get slow and blast radius gets scary. Split by **blast radius**:

- `platform/network` — VNets, VPCs, DNS.
- `platform/identity` — RBAC baselines, managed identities, service principals.
- `platform/observability` — Log Analytics, Event Hubs, centralized logging.
- `apps/<app>` — per‑application stack.

Share data across states via:

- Provider data sources (e.g., `data "azurerm_virtual_network"`) — preferred.
- `terraform_remote_state` data source — convenient but couples the consumer to the producer's output schema.
- HCP Terraform: `tfe_outputs` data source.

## Refactoring without destroying
These in‑config blocks beat raw `terraform state` commands because they live in version control and run on every `plan`/`apply`.

### `moved` (Terraform ≥ 1.1)
Rename or relocate resources and module calls without destroy/create:

```hcl path=null start=null
moved {
  from = aws_instance.a
  to   = aws_instance.web
}

moved {
  from = module.a
  to   = module.web
}
```

### `import` (Terraform ≥ 1.5)
Bring existing infra under management, in code:

```hcl path=null start=null
import {
  to = azurerm_resource_group.existing
  id = "/subscriptions/.../resourceGroups/rg-existing"
}

resource "azurerm_resource_group" "existing" {
  name     = "rg-existing"
  location = "eastus"
}
```

Prefer this to `terraform import` — the import appears in plans and PRs.

### `removed` (Terraform ≥ 1.7)
Stop managing a resource without destroying it:

```hcl path=null start=null
removed {
  from = azurerm_storage_account.legacy
  lifecycle {
    destroy = false
  }
}
```

### The CLI fallbacks
When in‑config blocks cannot express the change (e.g., moving between states):

- `terraform state mv SRC DST`
- `terraform state rm ADDR`
- `terraform state pull`/`push` — use with extreme caution.

Always back up state before running these: `terraform state pull > state.backup.$(date +%s).json`.

## State hygiene
- Enable versioning on the state storage (Azure Storage blob versioning, S3 versioning). It has saved many incidents.
- Encrypt at rest and require TLS.
- Restrict access to the storage account to the specific identities that run Terraform; nobody else.
- Never commit state or plan files. `.gitignore` rules: `*.tfstate`, `*.tfstate.*`, `.terraform/`, `.terraform.tfstate.lock.info`, `tfplan`, `*.tfplan`, `crash.log*`.
