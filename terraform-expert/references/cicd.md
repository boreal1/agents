# CI/CD for Terraform

The shape of the pipeline is the same everywhere:

1. **Lint** — `terraform fmt -check -recursive`, `tflint`.
2. **Init** — `terraform init` (with real backend config in CI).
3. **Validate** — `terraform validate`.
4. **Security scan** — `tfsec` or `checkov`.
5. **Plan** — `terraform plan -out=tfplan`, archive the plan as an artifact, post the summary on the PR.
6. **Manual gate** (for non‑dev envs) — a human approves the exact plan.
7. **Apply** — `terraform apply tfplan`. Never re‑plan at apply time.

Authenticate with **short‑lived, federated credentials** (OIDC / workload identity federation). Never long‑lived secrets.

## GitHub Actions → Azure via OIDC

Set up a Microsoft Entra ID app registration with a federated credential trusting `repo:<org>/<repo>:ref:refs/heads/main` (and pull_request, environment, etc. as appropriate). Grant that app the needed Azure RBAC.

```yaml path=null start=null
name: terraform

on:
  pull_request:
    paths: ['infra/**']
  push:
    branches: [main]
    paths: ['infra/**']

permissions:
  id-token: write        # required for OIDC
  contents: read
  pull-requests: write   # for plan comments

jobs:
  validate:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: infra/envs/prod
    steps:
      - uses: actions/checkout@v4

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.13.0   # pin; match .terraform-version
          terraform_wrapper: false

      - name: Azure login (OIDC)
        uses: azure/login@v2
        with:
          client-id:       ${{ vars.AZURE_CLIENT_ID }}
          tenant-id:       ${{ vars.AZURE_TENANT_ID }}
          subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}

      - run: terraform fmt -check -recursive
      - run: terraform init
      - run: terraform validate

      - name: tflint
        uses: terraform-linters/setup-tflint@v4
      - run: tflint --init && tflint

      - name: Plan
        if: github.event_name == 'pull_request'
        run: terraform plan -out=tfplan -input=false -no-color
      - uses: actions/upload-artifact@v4
        if: github.event_name == 'pull_request'
        with:
          name: tfplan-${{ github.sha }}
          path: infra/envs/prod/tfplan
          retention-days: 7

  apply:
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    needs: validate
    runs-on: ubuntu-latest
    environment: prod            # requires a reviewer per environment protection rule
    defaults:
      run:
        working-directory: infra/envs/prod
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.13.0
          terraform_wrapper: false
      - uses: azure/login@v2
        with:
          client-id:       ${{ vars.AZURE_CLIENT_ID }}
          tenant-id:       ${{ vars.AZURE_TENANT_ID }}
          subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}
      - run: terraform init
      - run: terraform plan -out=tfplan -input=false -no-color
      - run: terraform apply -input=false -auto-approve tfplan
```

Tips:

- Put the provider auth env vars `ARM_USE_OIDC=true`, `ARM_CLIENT_ID`, `ARM_TENANT_ID`, `ARM_SUBSCRIPTION_ID` in the job environment if you are not using the `azure/login` action — the `azurerm` provider will pick them up automatically.
- Use [GitHub Environments](https://docs.github.com/en/actions/deployment/targeting-different-environments/using-environments-for-deployment) for the manual approval gate. Required reviewers are set per environment, not per workflow.
- For multi‑env repos, a matrix strategy over `[dev, stg, prod]` with an environment name per matrix entry keeps one workflow file.

## Azure DevOps → Azure via workload identity federation

Azure DevOps now supports workload identity federation on service connections. Configure a service connection of type **Azure Resource Manager → Workload Identity federation (automatic)** and give its service principal the appropriate Azure roles.

```yaml path=null start=null
trigger:
  branches: { include: [main] }
  paths:    { include: [infra/**] }

pool:
  vmImage: ubuntu-latest

variables:
  workingDir: infra/envs/prod
  serviceConnection: sc-azure-prod   # workload-identity-federation
  tfVersion: 1.13.0

stages:
  - stage: validate
    jobs:
      - job: validate
        steps:
          - task: TerraformInstaller@1
            inputs: { terraformVersion: $(tfVersion) }
          - checkout: self
          - task: AzureCLI@2
            displayName: terraform init/validate/plan
            inputs:
              azureSubscription: $(serviceConnection)
              scriptType: bash
              addSpnToEnvironment: true
              workingDirectory: $(workingDir)
              scriptLocation: inlineScript
              inlineScript: |
                export ARM_USE_OIDC=true
                export ARM_CLIENT_ID=$servicePrincipalId
                export ARM_TENANT_ID=$tenantId
                export ARM_SUBSCRIPTION_ID=$(az account show --query id -o tsv)
                export ARM_OIDC_TOKEN=$idToken
                terraform fmt -check -recursive
                terraform init
                terraform validate
                terraform plan -out=tfplan -input=false -no-color
          - publish: $(workingDir)/tfplan
            artifact: tfplan

  - stage: apply
    dependsOn: validate
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
    jobs:
      - deployment: apply
        environment: prod        # approvals/checks configured on the environment
        strategy:
          runOnce:
            deploy:
              steps:
                - download: current
                  artifact: tfplan
                - task: TerraformInstaller@1
                  inputs: { terraformVersion: $(tfVersion) }
                - task: AzureCLI@2
                  inputs:
                    azureSubscription: $(serviceConnection)
                    scriptType: bash
                    addSpnToEnvironment: true
                    workingDirectory: $(workingDir)
                    scriptLocation: inlineScript
                    inlineScript: |
                      export ARM_USE_OIDC=true
                      export ARM_CLIENT_ID=$servicePrincipalId
                      export ARM_TENANT_ID=$tenantId
                      export ARM_SUBSCRIPTION_ID=$(az account show --query id -o tsv)
                      export ARM_OIDC_TOKEN=$idToken
                      cp $(Pipeline.Workspace)/tfplan tfplan
                      terraform init
                      terraform apply -input=false -auto-approve tfplan
```

## Universal rules
- Pin every tool version: Terraform CLI, providers, modules, `tflint`/`tfsec`/`checkov`. Floating versions are how pipelines break on Mondays.
- Save the plan; apply that exact plan. Do not re‑plan at apply time.
- One pipeline per environment, or a matrix job — do not guess the environment from branch name in shell scripts.
- Fail loud on `fmt` drift, `validate` errors, and high‑severity security findings. Auto‑merging on these is how regressions reach prod.
- Put lint/security/plan commentary back on the PR. A plan you cannot see is a plan you will approve blindly.
