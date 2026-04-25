# CI/CD Patterns for Kubernetes

Detailed pipeline patterns for GitOps tools and CI platforms. Read this when
you're actively building deployment pipelines.

## Table of Contents

1. [ArgoCD](#argocd)
2. [Flux](#flux)
3. [GitHub Actions](#github-actions)
4. [GitLab CI](#gitlab-ci)
5. [Argo Workflows](#argo-workflows)
6. [Image Tagging Strategy](#image-tagging-strategy)

## ArgoCD

### Application

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/myapp-deploy
    targetRevision: main
    path: charts/myapp
    helm:
      releaseName: myapp
      valueFiles:
        - values.yaml
        - values-prod.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: myapp
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

### ApplicationSet (multi-environment)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: myapp
spec:
  generators:
    - list:
        elements:
          - cluster: dev-aks
            url: https://dev-aks-api
            valuesFile: values-dev.yaml
          - cluster: staging-aks
            url: https://staging-aks-api
            valuesFile: values-staging.yaml
          - cluster: prod-aks
            url: https://prod-aks-api
            valuesFile: values-prod.yaml
  template:
    metadata:
      name: 'myapp-{{cluster}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/myorg/myapp-deploy
        targetRevision: main
        path: charts/myapp
        helm:
          valueFiles:
            - 'values.yaml'
            - '{{valuesFile}}'
      destination:
        server: '{{url}}'
        namespace: myapp
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

## Flux

### GitRepository + Kustomization

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: myapp-deploy
  namespace: flux-system
spec:
  interval: 1m
  url: https://github.com/myorg/myapp-deploy
  ref:
    branch: main
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: myapp
  namespace: flux-system
spec:
  interval: 5m
  path: ./environments/prod
  prune: true
  sourceRef:
    kind: GitRepository
    name: myapp-deploy
  targetNamespace: myapp
  wait: true
  timeout: 5m
```

### HelmRelease

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: myapp
  namespace: myapp
spec:
  interval: 5m
  chart:
    spec:
      chart: ./charts/myapp
      sourceRef:
        kind: GitRepository
        name: myapp-deploy
        namespace: flux-system
  values:
    replicaCount: 3
    image:
      tag: "1.2.3"
  install:
    remediation:
      retries: 3
  upgrade:
    remediation:
      retries: 3
```

## GitHub Actions

### Build, Push, and Update Manifest

```yaml
name: Build and Deploy

on:
  push:
    branches: [main]

permissions:
  contents: read
  id-token: write   # for Azure / AWS OIDC

env:
  REGISTRY: myregistry.azurecr.io
  IMAGE: myapp

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      image_tag: ${{ steps.meta.outputs.tag }}
    steps:
      - uses: actions/checkout@v4

      - name: Compute image tag
        id: meta
        run: echo "tag=${GITHUB_SHA::7}" >> $GITHUB_OUTPUT

      - name: Azure login (OIDC)
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: ACR login
        run: az acr login --name ${{ env.REGISTRY }}

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          push: true
          tags: ${{ env.REGISTRY }}/${{ env.IMAGE }}:${{ steps.meta.outputs.tag }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          platforms: linux/amd64,linux/arm64

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          repository: myorg/myapp-deploy
          token: ${{ secrets.DEPLOY_REPO_TOKEN }}

      - name: Update image tag
        run: |
          yq -i '.image.tag = "${{ needs.build.outputs.image_tag }}"' \
            charts/myapp/values-prod.yaml

      - name: Commit and push
        run: |
          git config user.name "github-actions"
          git config user.email "actions@github.com"
          git add .
          git commit -m "chore: bump myapp to ${{ needs.build.outputs.image_tag }}"
          git push
```

This is the standard GitOps pattern: CI builds the image and updates a manifest
repo. ArgoCD or Flux picks up the change and deploys. The CI never touches the
cluster directly.

### Helm Chart Test

```yaml
name: Helm Chart Test

on:
  pull_request:
    paths:
      - 'charts/**'

jobs:
  lint-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - uses: azure/setup-helm@v4

      - name: Helm lint
        run: helm lint charts/*

      - name: Helm template
        run: |
          for chart in charts/*; do
            helm template "$chart" --debug
          done

      - uses: helm/chart-testing-action@v2
      - name: Run chart-testing
        run: ct lint --target-branch main

      - uses: helm/kind-action@v1
      - name: Install charts
        run: ct install --target-branch main
```

## GitLab CI

```yaml
stages:
  - build
  - deploy

variables:
  REGISTRY: myregistry.azurecr.io
  IMAGE: $REGISTRY/myapp

build:
  stage: build
  image: docker:25
  services:
    - docker:25-dind
  before_script:
    - docker login -u $REGISTRY_USER -p $REGISTRY_PASSWORD $REGISTRY
  script:
    - export TAG=${CI_COMMIT_SHORT_SHA}
    - docker buildx build --platform linux/amd64 -t $IMAGE:$TAG --push .
    - echo "TAG=$TAG" > build.env
  artifacts:
    reports:
      dotenv: build.env

update-manifest:
  stage: deploy
  image: alpine/git:latest
  script:
    - git clone https://oauth2:$DEPLOY_TOKEN@gitlab.com/myorg/myapp-deploy.git
    - cd myapp-deploy
    - apk add --no-cache yq
    - yq -i ".image.tag = \"$TAG\"" charts/myapp/values-prod.yaml
    - git config user.email "ci@gitlab.com"
    - git config user.name "GitLab CI"
    - git commit -am "chore: bump myapp to $TAG"
    - git push
```

## Argo Workflows

For complex pipeline DAGs that run inside the cluster:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: build-deploy-
spec:
  entrypoint: pipeline
  arguments:
    parameters:
      - name: repo
        value: https://github.com/myorg/myapp
      - name: revision
        value: main

  templates:
    - name: pipeline
      dag:
        tasks:
          - name: clone
            template: git-clone
          - name: test
            template: run-tests
            dependencies: [clone]
          - name: build
            template: build-image
            dependencies: [test]
          - name: update-manifest
            template: bump-tag
            dependencies: [build]

    - name: git-clone
      container:
        image: alpine/git
        command: [sh, -c]
        args: ["git clone {{workflow.parameters.repo}} /workspace"]
        volumeMounts:
          - name: workspace
            mountPath: /workspace

    - name: build-image
      container:
        image: gcr.io/kaniko-project/executor:latest
        args:
          - --context=/workspace
          - --destination=myregistry.azurecr.io/myapp:{{workflow.parameters.revision}}
        volumeMounts:
          - name: workspace
            mountPath: /workspace

  volumes:
    - name: workspace
      emptyDir: {}
```

## Image Tagging Strategy

Never use `:latest` in production. Use one of:

- **Git SHA (short)** — `1a2b3c4` — most reproducible, easy to trace back
- **Semver + SHA** — `v1.2.3-1a2b3c4` — combines release version with traceability
- **Date + SHA** — `2026.04.25-1a2b3c4` — useful for time-based deployments

For development/staging, you can also use:
- **Branch + SHA** — `feature-auth-1a2b3c4`

Always pin the tag in the manifest. The pipeline computes the tag and writes it
to the manifest repo — never let the pulled tag drift between deployments.
