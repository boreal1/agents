# k8s-platform-engineer

A Warp skill for authoring production-grade Kubernetes infrastructure: manifests,
Helm charts, CI/CD pipelines, networking, and observability. Azure-first defaults
with full AWS/EKS coverage.

## What this skill does

Activated when the user mentions Kubernetes, Helm charts, AKS, EKS, ArgoCD, Flux,
GitOps, Cilium/Istio/Linkerd, cert-manager, Prometheus, the LGTM stack, or asks
for platform engineering work — even when they don't say "Kubernetes" explicitly
(e.g., "write a Helm chart for my API", "set up observability for my cluster").

The skill encodes the things a senior platform engineer would never forget:
production resource limits, health probes, pod disruption budgets, security
contexts, recommended labels, ConfigMap checksum annotations, default-deny
NetworkPolicies (with the DNS egress gotcha), cross-signal observability
correlation, and GitOps-first CI/CD that doesn't push directly to clusters.

## Layout

```
k8s-platform-engineer/
├── SKILL.md                       # core instructions, workflow, principles
├── README.md                      # this file
├── references/                    # progressive disclosure — loaded on demand
│   ├── helm-patterns.md           # _helpers.tpl, all template types, multi-env
│   ├── networking.md              # CNI, Cilium, Istio, Linkerd, Gateway API, NP
│   ├── observability.md           # Prom/Loki/Tempo/Grafana/OTel, alerting
│   └── cicd.md                    # ArgoCD, Flux, GHA, GitLab CI, Argo Workflows
└── evals/
    └── evals.json                 # 4 eval prompts with assertions
```

## Benchmark results (iteration 1)

Evaluated on 4 representative platform-engineering tasks. Each task was run
twice — once with the skill loaded into the agent's context (`with_skill`) and
once without (`without_skill`, baseline) — and graded against pre-defined
assertions.

| Configuration | Mean Pass Rate | Total Assertions |
|---|---|---|
| **with_skill** | **100.0%** | 30/30 |
| without_skill | 29.8% | 9/30 |
| **Delta** | **+70.2 pp** | — |

### Per-eval breakdown

| Eval | with_skill | baseline | Δ |
|---|---|---|---|
| eval-1-helm-chart (production-grade Helm chart) | 9/9 (100%) | 3/9 (33%) | +67 pp |
| eval-2-lgtm-bootstrap (LGTM observability) | 7/7 (100%) | 2/7 (29%) | +71 pp |
| eval-3-github-actions-gitops (GHA + ArgoCD) | 7/7 (100%) | 1/7 (14%) | +86 pp |
| eval-4-cilium-networking (Cilium + Hubble + NP) | 7/7 (100%) | 3/7 (43%) | +57 pp |

### Where the skill creates the most value

The largest single-assertion deltas show where domain expertise matters most:

- **`dns-egress-allowed`** (eval-4) — without explicit guidance, the agent
  forgets to allow DNS egress when applying NetworkPolicies. This is the
  single most common NP bug in production. The skill's `references/networking.md`
  flags it explicitly.
- **`cd-not-pushing-to-cluster`** (eval-3) — the baseline imperatively pushes
  to AKS via `azure/aks-set-context` + `helm upgrade`. The skill enforces
  the GitOps pull model: CI updates a manifest repo, the cluster reconciles.
- **`cross-signal-correlation`** (eval-2) — without the skill, observability
  setups install the four LGTM components but skip the trace-to-log and
  exemplar wiring that makes them actually useful together.
- **`runAsNonRoot` / `pdb-included` / `checksum-annotation`** (eval-1) —
  the omissions that distinguish a "works in dev" chart from a
  production-safe one.

### Failure mode

The baseline's failure mode is omission, not error. Every baseline output
is syntactically valid YAML that would deploy without crashing. The skill's
value is encoding the *what to include* checklist that a generic agent
otherwise glosses over.

## Eval workspace

Full eval outputs, per-run grading JSON, and the re-runnable grader live at
`.agents/skills/k8s-platform-engineer-workspace/iteration-1/`. Re-run with:

```bash
python3 .agents/skills/k8s-platform-engineer-workspace/iteration-1/grade.py
```

The grader uses a `yaml_blocks()` helper that extracts `\`\`\`yaml` fenced
code blocks (and any `.yaml`/`.yml`/`.tpl` files) so substring assertions
ignore prose and only consider deployable config.

## Pairing with terraform-expert

This skill is designed to pair with the existing `terraform-expert` skill.
Terraform provisions the AKS cluster, ACR, storage accounts, and identity;
this skill handles everything that runs *on* the cluster.
