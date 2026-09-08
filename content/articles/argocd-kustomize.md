---
date: "2026-06-04T00:00:00+09:00"
title: "ArgoCD with Kustomize: examples"
tags: ["Argo CD", "Kubernetes"]
---

Here are examples of how Kustomize can be used with Argo CD.

## Example 1: referencing a single file

Repo A contains a plain manifest. Repo B's `kustomization.yml` references that file directly and patches it.

### Repo A

```
repo-a
└── path/to
    └── deployment.yml
```

`deployment.yml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: test
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: nginx
  template:
    metadata:
      labels:
        app.kubernetes.io/name: nginx
    spec:
      containers:
        - name: nginx
          image: nginx
```

### Repo B

```
repo-b
└── path/to
    ├── kustomization.yml
    └── patch.yml
```

`kustomization.yml`

```yml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - https://github.com/your-org/repo-a//path/to/deployment.yml?ref=v1.0.0

patches:
  - path: ./patch.yml
```

NOTE: the `//` in the `resources` URL is intentional. It marks where the repo root ends and the in-repo path begins, e.g. `repo-a//path/to/deployment.yml`. This is the preferred, most portable syntax since it works for any git host.

On `github.com` specifically, Kustomize can also infer the boundary without `//`, since it assumes an org/repo is always 2 path segments (e.g. `your-org/repo-a/path/to/deployment.yml` would resolve the same way). Other markers Kustomize recognizes instead of `//` are a `.git` suffix (`repo-a.git/path/to/deployment.yml`) and, for Azure DevOps, `_git/`.

`patch.yml`

```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: test
spec:
  replicas: 2
```

ArgoCD application:

```yml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nginx
  namespace: argocd
spec:
  source:
    repoURL: https://github.com/your-org/repo-b
    targetRevision: main
    path: path/to
  destination:
    server: https://kubernetes.default.svc
    namespace: whatever
  project: default
```

## Example 2: referencing a directory

This time Repo A has its own `kustomization.yml`, so Repo B references the directory instead of the file directly. This is the more common pattern in practice — most published Kustomize bases (e.g. `kubernetes-csi/external-snapshotter`) work this way.

### Repo A

```
repo-a
└── path/to
    ├── deployment.yml
    └── kustomization.yml
```

`deployment.yml`: same as in Example 1.

`kustomization.yml`

```yml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yml
```

### Repo B

```
repo-b
└── path/to
    ├── kustomization.yml
    └── patch.yml
```

`kustomization.yml`

```yml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - https://github.com/your-org/repo-a//path/to?ref=v1.0.0

patches:
  - path: ./patch.yml
```

Note the `resources` URL now points at the directory (`.../path/to`) rather than the file, since Repo A's `kustomization.yml` already declares `deployment.yml` as a resource.

`patch.yml`: same as in Example 1.

The Argo CD `Application` manifest is unchanged from Example 1.
