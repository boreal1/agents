# Terraform style conventions

These are the idioms you should default to. They mirror [HashiCorp's published style guide](https://developer.hashicorp.com/terraform/language/style) — which is also available via the `terraform` MCP resource at `/terraform/style-guide` — plus field experience on medium‑to‑large Terraform codebases. If you find yourself fighting them, there is usually a reason worth saying out loud before you break the rule.

Where the official guide is unambiguous (meta‑argument placement, file names, parameter order), treat it as authoritative. Where it is soft ("use `count` and `for_each` sparingly"), the notes below add concrete guidance from real projects.

## File layout
A root module (or reusable child module) should separate concerns:

- `terraform.tf` — a single `terraform {}` block with `required_version` and `required_providers`. Nothing else.
- `providers.tf` — all `provider` blocks (including aliases). Default provider first, then aliases.
- `backend.tf` — the `terraform { backend "..." {} }` block (kept separate so you can template it per environment).
- `variables.tf` — all `variable` blocks, alphabetical.
- `locals.tf` — shared `locals` (file‑local locals can live at the top of their file).
- `main.tf` — resources and data sources for small modules. Split by domain (`network.tf`, `compute.tf`, `storage.tf`, etc.) as the module grows.
- `outputs.tf` — all `output` blocks, alphabetical.
- `override.tf` / `*_override.tf` — loaded last and override earlier definitions. Use sparingly, and leave a `#` comment on the original resource pointing at the override so maintainers know where to look.
- `README.md` — purpose, inputs, outputs, an example caller, and diagram if the module is nontrivial.
- `.terraform-version` / `.tool-versions` — pin the Terraform CLI version so collaborators and CI match.
- `.gitignore` — at minimum: `.terraform/`, `*.tfstate*`, `.terraform.tfstate.lock.info`, `*.tfvars` (unless you intentionally commit a non‑sensitive example), `crash.log*`, `tfplan`, `*.tfplan`.

Always commit `.terraform.lock.hcl`.

## Blocks and arguments
- Two‑space indentation. `terraform fmt` will enforce this — run it.
- Align `=` on consecutive single‑line arguments at the same nesting level (again, `fmt`).
- Inside a block body, put all arguments at the top, then nested blocks below them, with one blank line between the two sections. Use blank lines to separate logical groups of arguments within that top section.
- Top‑level blocks (`resource`, `data`, `module`, `variable`, `output`, …) are separated from each other by exactly one blank line.
- Meta‑arguments come either first or last — never sprinkled in the middle:
  - `count` / `for_each` first (they control how many instances exist).
  - `provider` with them at the top when you need an aliased provider.
  - `lifecycle` block and the `depends_on` argument last.
- Avoid grouping nested blocks of different types, except when the provider explicitly defines a family (e.g., `root_block_device`, `ebs_block_device`, `ephemeral_block_device` on `aws_instance`).
- Resource type and name in double quotes: `resource "azurerm_storage_account" "logs" { ... }`.
- Use `#` for comments; `//` and `/* */` work but are not idiomatic.

## Naming
- Resources, variables, outputs, locals: `snake_case`, descriptive nouns, no type redundancy.
  - ✅ `resource "aws_instance" "web_api"`
  - ❌ `resource "aws_instance" "web_api_aws_instance"`
- Module block names describe the thing produced, not the module source: `module "network" { source = "..." }`.
- Modules published to a registry must live in a repo named `terraform-<provider>-<name>` (e.g., `terraform-azurerm-landing-zone`).

## Resource order
The order resources appear in your files has no effect on Terraform's graph, so optimize for a human reader.

- **Build on itself.** Define a data source (or a resource) *before* the resource that consumes it. Reading top‑to‑bottom should feel like a dependency chain, not a scavenger hunt.
- **Group by domain as modules grow.** Once `main.tf` exceeds roughly 200 lines, split by concern (`network.tf`, `compute.tf`, `storage.tf`, …) so a maintainer can jump straight to the file relevant to their change.

Inside a single `resource` / `data` / `module` block, follow this parameter order — `terraform fmt` does not enforce it but `tflint` and code review should:

1. `count` / `for_each` meta‑argument (if present).
2. `provider` meta‑argument (if using an aliased provider).
3. Non‑block arguments (strings, numbers, lists, maps).
4. Nested blocks (`network_interface`, `ebs_block_device`, …).
5. `lifecycle` block (if present).
6. `depends_on` argument (if present).

## Variables
- Always declare `type` and `description`.
- Provide a `default` only if the input is optional.
- Mark secrets with `sensitive = true` (does not encrypt state — do not confuse yourself).
- Use `validation` blocks when the type system cannot express the constraint (regex, numeric ranges, enum‑like sets). Keep error messages actionable.
- Prefer complex object types to sprawling flat lists of variables: `variable "network" { type = object({ cidr = string, subnets = map(object({...})) }) }` groups related inputs cohesively.
- **Do not over‑expose.** Before adding a new variable, ask whether the value will actually differ between deployments. Values that never change (e.g., a VM image family the workload was designed around, a fixed port number) are harder to read as variables than as literals — inline them and leave a comment if it is non‑obvious.
- Parameter order inside each `variable` block: `type`, `description`, `default`, `sensitive`, `validation`.

## Outputs
- Every output has a `description`.
- Mark sensitive outputs. They are still written to state in plaintext but will not be printed.
- Parameter order: `description`, `value`, `sensitive`.

## Locals
- Use sparingly. A local that is used once is usually just noise — inline it.
- Use them for non‑trivial computed values, naming conventions, and to simplify nested expressions. A good test: "Would naming this make the call site read better?"

## Providers
- Always declare a default (unaliased) provider.
- Define aliases in the same file as the default. Put the default first.
- For modules, never put `provider {}` blocks inside the module. Declare `required_providers` (with `configuration_aliases` if you need aliases) and accept providers explicitly via the caller's `providers = { ... }` map. This keeps `count`/`for_each`/`depends_on` working on `module` blocks.

## `for_each` vs `count`
The official guide says to use both "sparingly" — meta‑arguments add cognitive load, so reach for them only when they eliminate real duplication.

- `for_each` (map or set of strings) when each instance has a distinct identity you care about. Indices are stable across adds/removes.
- `count` when you genuinely want positional instances or a simple 0/1 toggle:
  ```hcl path=null start=null
  count = var.enable_metrics ? 1 : 0
  ```
- Avoid `count = length(var.some_list)` on real resources — reorders cause churn. Use `for_each = toset(var.some_list)`.
- If the effect of the meta‑argument is not obvious at a glance, leave a one‑line `#` comment. Future you will thank you.

## `dynamic` blocks
- Fine for truly repeated nested blocks driven by input (`tags`, `ingress` rules). Do not reach for `dynamic` as the default — static blocks are easier to read.

## Dependencies
- Prefer implicit dependencies via resource attribute references.
- Use `depends_on` only when Terraform genuinely cannot infer the dependency (IAM propagation, side‑effects behind a data source). Comment why.

## Refactoring
- Prefer `moved { from = ..., to = ... }` blocks (Terraform ≥ 1.1) over `terraform state mv`. They version‑control the refactor.
- For deletions you do not want Terraform to actually destroy, use `removed {}` blocks (Terraform ≥ 1.7) with `lifecycle { destroy = false }`.
- For importing existing infra, use `import {}` blocks (Terraform ≥ 1.5) in preference to `terraform import` — you get it in code review.

## Comments
Explain *why*, not *what*. A comment that paraphrases the next line is noise; a comment that explains why we chose ZRS instead of GRS is gold.
