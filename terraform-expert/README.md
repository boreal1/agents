# terraform-expert

A Warp skill that gives the agent expert-level Terraform knowledge with a strong
bias toward Azure + Azure Verified Modules, aligned with the Microsoft Cloud
Adoption Framework (CAF) and Azure Landing Zones (ALZ).

Key behaviors:

- Grounds every module/provider/resource choice in the `terraform` MCP server
  rather than memorized versions.
- Proactively checks for deprecated modules, providers, and resource arguments.
- Defaults to `Azure/avm-res-*` and `Azure/avm-ptn-*` modules when writing
  Azure infrastructure, and to the CAF-aligned pattern stack
  (`avm-ptn-alz`, `avm-ptn-alz-management`, `avm-ptn-hubnetworking`,
  `avm-ptn-alz-sub-vending` — which supersedes the deprecated
  `Azure/lz-vending/azurerm` — and `Azure/naming/azurerm`) for landing zones.
- Always finishes by running `terraform fmt`, `terraform init`, and
  `terraform validate` — a change is not done until all three pass.

## Layout

```
terraform-expert/
├── SKILL.md                         # triggering + core workflow
├── README.md                        # this file
└── references/
    ├── caf.md                       # Cloud Adoption Framework + Azure Landing Zones
    ├── avm.md                       # Azure Verified Modules deep dive
    ├── style.md                     # HCL style + file layout conventions
    ├── validation.md                # fmt/init/validate + lint/security tooling
    ├── state-and-backends.md        # remote state, isolation, moved/import/removed
    └── cicd.md                      # GitHub Actions + Azure DevOps pipelines (OIDC)
```

The agent loads `SKILL.md` every time the skill triggers and pulls reference
files in on demand to keep context small.
