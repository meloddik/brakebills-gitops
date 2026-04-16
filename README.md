# brakebills-gitops

Workload and cluster-add-on manifests for the **brakebills-talos** cluster.

Argo CD runs on the cluster and watches this repo. Anything under
`clusters/<cluster-name>/apps/` becomes a child `Application`, and each child
points at the content it should render (an upstream Helm chart, a kustomize
tree, or a directory of raw manifests) — the app-of-apps pattern.

The cluster itself (Talos machine config, Terraform, Argo CD **bootstrap**)
lives in the sibling platform repo
[brakebills-talos](https://github.com/meloddik/brakebills-talos). That repo
runs **once** per cluster lifetime; this one runs continuously via Argo CD.

## Layout

```
clusters/
└── brakebills/
    ├── apps/            # Argo CD Application manifests (the app-of-apps roots)
    │   ├── README.md
    │   └── argocd.yaml  # Argo CD self-managing itself from the upstream chart
    └── argocd/          # values / manifests referenced by apps/argocd.yaml
        └── values.yaml  # Helm values — source of truth for the argocd install
```

Each app gets a sibling directory under `clusters/brakebills/` holding its
supporting content (chart values, kustomize overlays, raw manifests) and a
matching Application file in `clusters/brakebills/apps/` that references it.

## Adding a new app

1. Decide the source type: **upstream Helm chart**, **kustomize tree**, or
   **raw manifests in a git path**.
2. For Helm / kustomize: create `clusters/brakebills/<appname>/` and put the
   supporting content there (`values.yaml`, `kustomization.yaml`, etc.).
3. Create `clusters/brakebills/apps/<appname>.yaml` — an Argo CD
   `Application` that points at the source.
4. Commit + push to `main`. The root Application on the cluster picks it up
   on the next sync (every ~3 minutes by default; use the Argo CD UI's "Sync"
   button for immediate reconcile).

## First-time bootstrap

Before this repo does anything useful, the cluster needs to exist and
Argo CD needs to be installed on it. That happens in the platform repo:

```bash
# In brakebills-talos
./gradlew deployCluster -Pcluster=brakebills   # Talos cluster via Terraform
./gradlew argoBootstrap -Pcluster=brakebills   # Argo CD install + root Application
```

`argoBootstrap` applies the root Application, which points at
`clusters/brakebills/apps/` in this repo. On first sync, it creates the
`argocd` child Application (`apps/argocd.yaml`), which takes ownership of
the running Argo CD — from then on, **upgrades happen by committing here**,
not by re-running `argoBootstrap`.

## Self-management invariant

The `argocd` child Application in `apps/argocd.yaml` uses the same upstream
Helm chart (`argoproj/argo-helm` → `argo-cd`) and the values file at
`clusters/brakebills/argocd/values.yaml` that `brakebills-talos` used to
bootstrap. If the two diverge — different chart version, different values,
different fullnameOverride — Argo CD will see drift on every sync and
reconcile itself in a loop. Keep them aligned:

- Chart version in `brakebills-talos`: `gitops-bootstrap/argocd/chart.env`
- Chart version here: `clusters/brakebills/apps/argocd.yaml` (`targetRevision`)
- Values file in `brakebills-talos`: `gitops-bootstrap/argocd/values.yaml`
- Values file here: `clusters/brakebills/argocd/values.yaml`

Until you feel comfortable deleting the bootstrap copies in the platform
repo, treat the platform repo as the disaster-recovery path and this repo as
the steady-state path.
