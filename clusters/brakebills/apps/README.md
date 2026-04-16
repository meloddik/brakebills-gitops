# App-of-apps root

Every `*.yaml` file in this directory is an Argo CD `Application`. The root
`Application` configured by `brakebills-talos` watches this directory
recursively, so each file here becomes a child `Application` in the cluster.

**Do not put raw workload manifests here.** Workload content lives in sibling
directories under `clusters/brakebills/<appname>/` and is referenced by an
`Application` file in this directory.

## Sync ordering

Argo CD applies everything in this directory as one sync wave unless you use
sync waves explicitly. If you need ordering (e.g. cert-manager CRDs before
anything that uses them), annotate the `Application`:

```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "-10"
```

Lower numbers apply earlier. Default is 0.
