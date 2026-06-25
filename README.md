# multicloud-karmada-gitops

Multi-cloud GitOps platform — **ArgoCD** + **Karmada** + **Helm** across **AWS + GCP** Kubernetes clusters.

A centralized cluster registry is the single source of truth. Applications reference clusters **only by alias**; all infrastructure data (region, VPC, identity) lives in one place and is projected onto ArgoCD cluster Secrets at runtime.

---

## Repository layout

```
karmada-gitops/
├── values.yml                         # GLOBAL CLUSTER REGISTRY (source of truth)
│
├── argocd/
│   ├── argocd.yml                     # ROOT entrypoint: root + cluster-registry + routing
│   └── cluster-registry/              # Helm chart: renders cluster Secrets from values.yml
│       ├── Chart.yaml
│       └── templates/cluster-secret.yaml
│
├── base/                              # platform add-ons (every / most clusters)
│   ├── aws-lb-controller/
│   ├── external-secrets/
│   └── karpenter/
│
├── aws-clusters/                      # AWS-targeted workloads
│   └── redis/
│
└── gcp-clusters/                      # GCP-targeted workloads
```

Each app folder is **data only**:

```
<app>/
├── value.yml            # app config: appName, namespace, clusterAliases, chart coords
├── chart.yml            # Helm metadata stub
├── values/node.yml      # app-generic Helm value overrides (NO cluster data)
└── templates/argocd.yml # per-app ApplicationSet (fans out by alias)
```

---

## Architecture rules

| File | Holds | Never holds |
|------|-------|-------------|
| `values.yml` | cluster defs, VPC, region, identity, **aliases** | app config |
| `<app>/value.yml` | alias references + chart coords + namespace | raw cluster details (region/VPC) |
| `<app>/chart.yml` | static Helm metadata | cluster logic / values / env |
| `<app>/values/node.yml` | app-generic Helm values | per-cluster values |
| `<app>/templates/argocd.yml` | ArgoCD ApplicationSet | hardcoded infra data |

---

## How it works

```
values.yml ──(cluster-registry chart)──▶ ArgoCD cluster Secrets
                                          (labels: cluster-alias, cloud, environment;
                                           annotations: cluster-name/region/vpc-id/role-arn)
                                                    │
 <app>/templates/argocd.yml (ApplicationSet)        │ clusters generator
   ├─ selects clusters by alias ◀───────────────────┘
   └─ source 1: git ($values)  + source 2: upstream Helm chart
        └─ valuesObject injects per-cluster data from Secret annotations
```

1. **Registry → Secrets.** The `cluster-registry` Application reads `values.yml` and renders one ArgoCD cluster Secret per alias, projecting registry data onto Secret labels/annotations. `sync-wave: -10` so clusters register before any app syncs.
2. **Routing.** The `routing` ApplicationSet walks `base/`, `aws-clusters/`, `gcp-clusters/` and applies only `*/templates/argocd.yml` (include filter skips data files).
3. **Per-app fan-out.** Each app's ApplicationSet `clusters` generator selects targets by `cluster-alias` label, then deploys the upstream chart with `node.yml` overrides + per-cluster values injected via `valuesObject`.

---

## Cluster registry (`values.yml`)

| alias | cloud | env | region | identity |
|-------|-------|-----|--------|----------|
| `prod-aws` | aws | prod | us-east-1 | IRSA (role-arn) |
| `dr-aws` | aws | dr | us-west-2 | IRSA (role-arn) |
| `dev-gcp` | gcp | dev | us-central1 | Workload Identity (gsa) |

---

## Applications

| App | Layer | Target aliases | Upstream chart |
|-----|-------|----------------|----------------|
| aws-lb-controller | base | prod-aws, dr-aws | `aws-load-balancer-controller` 1.7.2 |
| external-secrets | base | prod-aws, dr-aws, dev-gcp | `external-secrets` 0.9.20 |
| karpenter | base | prod-aws, dr-aws | `karpenter` 0.37.0 (OCI) |
| redis | aws-clusters | prod-aws, dr-aws | `bitnami/redis` 19.6.4 |

---

## Bootstrap

```bash
# 1. Install ArgoCD (multi-source enabled) on the management cluster.
# 2. Apply the root entrypoint — pulls everything else via GitOps.
kubectl apply -f karmada-gitops/argocd/argocd.yml
```

### Prerequisites

- **Cluster credentials** — `cluster-registry` writes only non-secret connection shape. bearerToken / CA must be injected into each `cluster-<alias>` Secret via `external-secrets` (not committed to git).
- **OCI Helm** — register `public.ecr.aws/karpenter` in ArgoCD with `enableOCI: true`.
- **GCP identity** — `external-secrets` on `dev-gcp` uses Workload Identity, not IRSA; ensure the registry projects the right identity annotation for that target.

---

## Add a new application

1. Create `<layer>/<app>/` with `value.yml`, `chart.yml`, `values/node.yml`, `templates/argocd.yml`.
2. Set `clusterAliases` to existing aliases from `values.yml`.
3. Commit. The `routing` ApplicationSet picks it up automatically — no edits to `argocd/argocd.yml`.

## Add a new cluster

1. Add a block under `clusters:` in `values.yml` (alias, cloud, region, vpc, identity, server).
2. Reference its alias from any app's `clusterAliases`.

---

## Roadmap

- **Karmada** — promote per-cluster ApplicationSet fan-out to Karmada propagation policies for centralized multi-cluster scheduling.
