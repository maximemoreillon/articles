+++
date = '2026-06-04T00:00:00+09:00'
title = "Use case for Kustomize with ArgoCD"
tags = ['Argo CD', 'Kubernetes']
+++

Here is an example of how Kustomize can be used with Argo CD.

- Repo A: contains `deployment.yml`
- Repo B: contains
  - `kustomization.yml`
  - `patch.yml`

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

`kustomization.yml`

```yml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - https://github.com/your-org/repo-a//path/to/deployment.yml?ref=v1.0.0

patches:
  - path: ./patch.yml
```

NOTE: double slash in `resources` URL is intentional

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
    path: path/to/kustomization.yml
  destination:
    server: https://kubernetes.default.svc
    namespace: whatever
  project: default
```
