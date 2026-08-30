---
date: "2026-08-26"
title: "Kustomize with GitLab CICD"
tags: ["GitLab", "DevOps", "Kubernetes"]
---

Kustomize is a convenient tool when it comes to deploying applications to multiple environments, where each has its own minor differences.
This article presents an approach to leverage Kustomize in a GitLab CICD pipeline.
Throughout this article, git tags deploy to the `prod` environment while the `staging` branch deploys to the eponymous environment.

## Repository structure

Here is an example repository structure for an application that would have both staging and production environments to deploy to:

```text
my-app/
├── kustomize/
│   ├── base/
│   │   ├── deployment.yml
│   │   └── kustomization.yml
│   └── overlays/
│       ├── staging/
│       │   └── kustomization.yml
│       └── prod/
│           └── kustomization.yml
├── ...
└── .gitlab-ci.yml
```

## Kustomize

In such a scenario, Kustomize would be used as follows.

### Base

In this example, a simple deployment manifest is applied by `kubectl`.

```yml
# kustomize/base/deployment.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: my-app
  template:
    metadata:
      labels:
        app.kubernetes.io/name: my-app
    spec:
      containers:
        - name: my-app
          image: my-registry/my-app:${IMAGE_TAG}
```

Note that here, the image tag is written as a shell variable (`${IMAGE_TAG}`) as it will be inferred from CICD and substituted with `envsubst`.

The base also needs its own `kustomization.yml` listing the resources the overlays will build on:

```yml
# kustomize/base/kustomization.yml
resources:
  - deployment.yml
```

### Overlays

To each deployment environment corresponds an overlay. In this case, `prod` simply applies the base manifest.

```yml
# kustomize/overlays/prod/kustomization.yml
resources:
  - ../../base
```

```yml
# kustomize/overlays/staging/kustomization.yml
resources:
  - ../../base

nameSuffix: -staging

labels:
  - pairs:
      app.kubernetes.io/name: my-app-staging
    includeSelectors: true
```

Because `includeSelectors: true` rewrites the Deployment's `spec.selector`, apply this overlay to a fresh Deployment: selector labels are immutable, so adding it to an already-deployed workload requires deleting and recreating it.

## CICD configuration

By leveraging `.gitlab-ci.yml`'s `rules` field, one can define per-branch - and thus per environment - variables.
This can be used to not only select the appropriate overlay, but also set the container image tag, which is substituted using `envsubst`.
In this example, `prod` uses the git tag as image tag while `staging` simply uses the commit sha. However, more environments can easily be added by creating dedicated branches and overlays, and adding an `if` statement to the rules.

```yml
# .gitlab-ci.yml
deploy-job:
  stage: deploy
  variables:
    TARGET_ENV: $CI_COMMIT_BRANCH
    IMAGE_TAG: $CI_COMMIT_SHORT_SHA
  rules:
    - if: $CI_COMMIT_TAG
      variables:
        TARGET_ENV: prod
        IMAGE_TAG: $CI_COMMIT_TAG
    - if: $CI_COMMIT_BRANCH == "staging"
  script:
    - >
      kubectl kustomize kustomize/overlays/${TARGET_ENV}
      | envsubst
      | kubectl apply -f -
```

The job image must provide both `kubectl` and `envsubst` (the latter ships with the `gettext` package). Cluster access is handled by the GitLab agent for Kubernetes.
