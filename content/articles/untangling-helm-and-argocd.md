---
date: "2026-09-08T00:00:00+09:00"
title: "Untangling ArgoCD and Helm"
tags: ["Kubernetes", "Helm", "ArgoCD"]
---

When getting ArgoCD to take over applications originally managed by Helm, the latter might still have those applications registered, meaning the `helm list` output would not reflect what the Kubernetes cluster actualy contains.

For example:

```
$ helm list -n cert-manager
NAME          NAMESPACE     REVISION	UPDATED                                	STATUS  	CHART                 APP VERSION
cert-manager  cert-manager  1       	2025-05-04 11:22:41.771600434 +0900 JST	deployed	cert-manager-v1.17.2  v1.17.2
```

While in fact

```
$ kubectl get applications -n argocd cert-manager -o jsonpath={.spec.source.targetRevision}
v1.19.4
```

De-registering an application from Helm can be achieved simply be removing the corresponding secret:

```
$ kubectl delete secret -n cert-manager -l owner=helm,name=cert-manager
```
