---
date: "2026-03-15"
title: "ArgoCD Helm chart with values from another repo"
tags: ["Kubernetes", "Helm", "ArgoCD"]
---

ArgoCD's multi-source applications make it possible to pull a Helm chart from one repo (e.g. its official Helm repository) while sourcing the `values.yaml` from another repo/path, such as a private repo holding cluster-specific configuration.

This is done by declaring two entries under `sources`: one for the chart itself, and one `ref` source pointing at the repo/path containing the values file. The chart source then references that values file via the `$<ref>/<path>` syntax.

For example, an application for `ingress-nginx` where the chart comes from the project's own Helm repo but the values come from a separate homelab configuration repo:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: ingress-nginx
  namespace: argocd
spec:
  project: default
  destination:
    server: https://kubernetes.default.svc
    namespace: ingress-nginx
  sources:
    - repoURL: https://kubernetes.github.io/ingress-nginx
      chart: ingress-nginx
      targetRevision: 4.15.1
      helm:
        valueFiles:
          - $values/ingress-nginx/values.yml
    - repoURL: git@github.com:maximemoreillon/homelab-k8s.git
      targetRevision: HEAD
      ref: values
```

The `ref: values` on the second source gives it a name that can be referenced elsewhere in the manifest. The first source then points `helm.valueFiles` at `$values/ingress-nginx/values.yml`, which resolves to `ingress-nginx/values.yml` inside that second repo.

This keeps chart versions decoupled from values, and allows values for many applications to live together in a single configuration repo instead of being duplicated per-application or baked into the chart source itself.
