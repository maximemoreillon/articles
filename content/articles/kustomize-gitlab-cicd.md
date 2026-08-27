---
date: "2026-08-26"
title: "Kustomize with GitLab CICD"
tags: ["GitLab", "DevOps", "Kubernetes"]
---

Kustomize is a convenient tool when it comes ro deploying applications to multiple environments, where each has its own minor differences.
This articles presents an approach to leverage Kustomize in GitLab CICD pipeline.
Throughout this article, the `master` branch is used to deploy to `prod` while `staging` deploys to the eponymous environment.

## Repository structure

Heres is an example repository structure for an application that would have both staging and production environments to deploy to:

```
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

In such Scenario, Kustomize would be used as follows.

### Base

In this example, a simple deployment manifest is applied by `kubectl`.

```yml
# repo/kustomize/base/deployment.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-app
          image: my-registry/my-app:{IMAGE_TAG}
```

Note that here, the image tag is set as environment variable as it will be infered from CICD.

### Overlays

To each deployment environment corresponds an overlay. In this case, `prod` simply applies the base manifest

```yml
# repo/kustomize/base/prod/kustomization.yml
resources:
  - ../../base
```

```yml
# repo/kustomize/base/staging/kustomization.yml
resources:
  - ../../base

nameSuffix: -staging

labels:
  - pairs:
      app.kubernetes.io/name: my-app-staging
    includeSelectors: true
```

## CICD configuration

By leveraging `.gitlab-ci.yml`'s `rules` field, one can define per-branch - and thus per environment - variables.
This can be used to not only select the appropriate overlay, but also set the container image tag, which is substituted using `envsubst`.
In this example, `prod` uses git tags as image tag while `staging` simply uses the commit sha.

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
