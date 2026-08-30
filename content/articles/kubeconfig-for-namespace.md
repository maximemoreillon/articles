---
date: "2026-08-24"
title: "Kubeconfig for specific namespace"
tags: ["Kubernetes", "Tutorials"]
---

Kubeconfig files allow clients such as `kubectl` or Headlamp to connect and authenticate to Kubernetes clusters so as to perform administrative and maintenance operations. Most often, cluster administrators get a kubeconfig file with cluster-wide administrator privileges from the Kubernetes distribution. For example, with Talos, this is achieved with `talosctl kubeconfig`.

In some cases, one would want to provide a kubectl file where privileges are limited to the scope of a single namespace. Examples would include providing a user with their own namespace that they can manage by themselves.

Doing so starts with the creation of a `ServiceAccount` object in the target namespace:

```yml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: namespace-limited-user
  namespace: your-target-namespace
```

The account can then be bound to a `ClusterRole` granting it administration privileges to the target namespace.

```yml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: namespace-limited-binding
  namespace: your-target-namespace
subjects:
  - kind: ServiceAccount
    name: namespace-limited-user
    namespace: your-target-namespace
roleRef:
  kind: ClusterRole
  name: admin
  apiGroup: rbac.authorization.k8s.io
```

Here, `admin` can be replaced with `view` for read-only permissions.

A kubeconfig file must be able to authenticate the client to the cluster. This is achieved with a token, created in Kubernetes as a `Secret` object:

```yml
apiVersion: v1
kind: Secret
metadata:
  name: namespace-limited-token
  namespace: your-target-namespace
  annotations:
    kubernetes.io/service-account.name: "namespace-limited-user"
type: kubernetes.io/service-account-token
```

Once created, the token can be extracted from the secret using:

```bash
kubectl \
  -n your-target-namespace get secret namespace-limited-token \
  -o jsonpath='{.data.token}' \
  | base64 --decode
```

A kubeconfig file also needs a certificate authority which can also be extracted from the secret:

```bash
kubectl \
  -n your-target-namespace get secret namespace-limited-token \
  -o jsonpath='{.data.ca\.crt}'
```

With this information, the kubeconfig file can be assembled:

```yml
apiVersion: v1
clusters:
  - cluster:
      certificate-authority-data: <BASE64_ENCODED_CA_CRT_OUTPUT>
      server: <CLUSTER_ENDPOINT_URL>
    name: my-cluster
contexts:
  - context:
      cluster: my-cluster
      user: namespace-limited-user
    name: my-limited-context
current-context: my-limited-context
kind: Config
preferences: {}
users:
  - name: namespace-limited-user
    user:
      token: <DECODED_TOKEN_STRING>
```
