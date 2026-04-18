# Validation: the exact commands and how to read their output

Validation is non‑negotiable. A Terraform change is not "done" until `fmt`, `init`, and `validate` all succeed. This file covers what each command actually checks, how to interpret failures, and the lightweight additional tooling worth running on top.

## The minimum viable validation loop

From the module or root directory:

```bash
terraform fmt -recursive
terraform init -backend=false
terraform validate
```

### `terraform fmt -recursive`
Rewrites files to the canonical HCL style. It is safe and idempotent. If it modifies anything on a supposedly finished change, that is a signal the code was hand‑formatted — run it before you commit, not after.

Use `terraform fmt -check -recursive` in CI: it exits non‑zero without modifying files, which is what you want on PRs.

### `terraform init -backend=false`
Downloads providers and child modules and writes `.terraform.lock.hcl`. The `-backend=false` flag tells it *not* to initialize the remote state backend — useful for validation in a sandbox with no cloud credentials.

Common failures and what they mean:

- `Error: Failed to query available provider packages` — typo in `source` (`hashicorp/azurerm`, not `hashicorp/AzureRM`), or a version constraint that no published release satisfies. Recheck via `get_latest_provider_version`.
- `Error: Module not installed` after editing `source` on a `module` block — re‑run `init` (or `init -upgrade` if you changed the `version` constraint).
- `Error: Inconsistent dependency lock file` — `required_providers` was changed without refreshing the lock file. Run `terraform init -upgrade` and commit the updated `.terraform.lock.hcl`.

When you need real backend access (CI, ops work), drop `-backend=false` and supply backend config via `-backend-config=...` or environment variables. Never commit backend config with credentials in it.

### `terraform validate`
Checks syntax, references, and type correctness. It does **not** contact any provider — so it will not catch things like "this region does not support this SKU". What it does catch:

- Undeclared variables, undefined references.
- Wrong types passed to arguments.
- Arguments that do not exist on the resource (great for catching deprecated/renamed arguments).
- Cyclic dependencies.

If `validate` passes but `plan` fails, the error is provider‑level (invalid value, auth, quota). `validate` has done its job.

## Useful additions
These are not mandatory, but a real CI pipeline should have at least two of them.

- **TFLint** (`tflint --init && tflint`) — static analysis for provider‑specific lint rules. The AzureRM ruleset catches deprecated arguments better than `validate` does. Pin a `.tflint.hcl` in the repo.
- **tfsec** or **Checkov** — security/compliance scanning. Both understand AVM modules (Checkov better, currently). Run on PRs and fail on `HIGH`/`CRITICAL`.
- **terraform-docs** — auto‑generates input/output tables for `README.md`. Wire it to a pre‑commit hook.
- **pre‑commit** — an easy way to run `terraform fmt -check`, `terraform validate`, `tflint`, `tfsec`, and `terraform-docs` on every commit locally. Mirror the same checks in CI.

A minimal `.pre-commit-config.yaml`:

```yaml path=null start=null
repos:
  - repo: https://github.com/antonbabenko/pre-commit-terraform
    rev: v1.96.2   # verify latest
    hooks:
      - id: terraform_fmt
      - id: terraform_validate
      - id: terraform_tflint
      - id: terraform_docs
      - id: terraform_tfsec
```

## When you actually want `plan`
`plan` talks to the provider, so it needs credentials and it will read remote state. Use it when:

- Reviewing someone else's change and you want to see what would happen.
- Before any `apply`. Always save the plan: `terraform plan -out=tfplan`, then `terraform show tfplan` to review, then `terraform apply tfplan`. Never re‑plan at apply time — the inputs could change.

Never run `terraform apply` without a saved plan in CI. Interactive `apply` from a workstation is fine for personal sandboxes; for shared environments, apply only plans reviewed by a human.
