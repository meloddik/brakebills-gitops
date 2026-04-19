# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Purpose

GitOps workload repo for the brakebills-talos cluster. Argo CD watches `clusters/brakebills/apps/` and reconciles child Applications from here. The cluster itself (Talos, Terraform, bootstrap) lives in the sibling repo [brakebills-talos](https://github.com/meloddik/brakebills-talos).

This repo is the source of truth for everything running on the cluster after bootstrap. Changes here are applied automatically by Argo CD — no Gradle commands needed.

## Layout

```
clusters/brakebills/
├── apps/                 # Argo CD Application manifests (app-of-apps roots)
│   ├── argocd.yaml       # self-managing Argo CD
│   ├── cilium.yaml       # Cilium CNI chart
│   ├── cilium-config.yaml # L2 policy + IP pool
│   ├── cert-manager.yaml # cert-manager chart
│   ├── cluster-issuer.yaml # Let's Encrypt ClusterIssuer
│   ├── external-dns.yaml # external-dns chart
│   └── argocd-ingress.yaml # Argo CD Ingress
├── argocd/               # values.yaml + ingress for argocd Application
├── cilium/               # values.yaml + L2/IP pool for cilium Application
├── cert-manager/         # values.yaml + ClusterIssuer manifest
└── external-dns/         # values.yaml for external-dns Application
```

## Adding a new app

1. Create `clusters/brakebills/<appname>/` with supporting content (values.yaml, manifests, etc.).
2. Create `clusters/brakebills/apps/<appname>.yaml` — an Argo CD Application pointing at the source.
3. Commit, push to `main`. Argo CD picks it up on the next sync (~3 minutes).

## Sync wave ordering

Lower numbers sync first. Current assignments:

| Wave | Application |
|------|------------|
| -30 | Cilium |
| -29 | cilium-config (L2 policy + IP pool) |
| -25 | cert-manager |
| -24 | ClusterIssuer |
| -23 | external-dns |
| -20 | argocd (self-managing) |
| -19 | argocd-ingress |
| 0+ | future workloads |

## Git workflow — strict gitflow

This repo uses **strict gitflow**. No exceptions, no shortcuts, no pushing directly to `main`.

### Branches

- **`main`** — production. Argo CD syncs from here. Every commit is a tagged release. Never commit directly.
- **`develop`** — integration. Features merge here first.
- **`feature/*`** — new work. Branches off `develop`, merges back to `develop`.
- **`release/*`** — release prep. Branches off `develop`, merges to `main` + back to `develop`. Tag on `main`.
- **`hotfix/*`** — urgent production fix. Branches off `main`, merges to `main` + `develop`. Tag on `main`.

### Feature flow (new work)
```
git checkout develop && git checkout -b feature/<name>
# work + commit (conventional commits)
git push -u origin feature/<name>
# merge feature -> develop
git checkout develop && git merge feature/<name> && git push origin develop
# cut release
git checkout -b release/vX.Y.Z develop
git checkout main && git merge release/vX.Y.Z
git tag -a vX.Y.Z -m "vX.Y.Z — summary"
git push origin main --tags
# merge back to develop
git checkout develop && git merge main && git push origin develop
```

### Hotfix flow (urgent production fix)
```
git checkout main && git checkout -b hotfix/<name>
# fix + commit
git checkout main && git merge hotfix/<name>
git tag -a vX.Y.Z -m "vX.Y.Z — fix description"
git push origin main --tags
# merge back to develop
git checkout develop && git merge main && git push origin develop
```

### Rules

- **Never push directly to `main`** — always via `release/*` or `hotfix/*`.
- **Never push directly to `develop`** — always via `feature/*` or back-merge from `main`.
- All commit messages use **Conventional Commits** (`feat:`, `fix:`, `refactor:`, `chore:`, `docs:`).
- Tags use **semver** (`vX.Y.Z`).
- **Argo CD syncs from `main` only.** Nothing on `develop` or feature branches affects the cluster.
