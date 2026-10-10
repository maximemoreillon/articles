---
date: "2026-09-20T00:00:00+09:00"
title: "Exposing Envoy Gateway on ports 80 and 443 with hostPort"
tags: ["Kubernetes", "Homelab", "Envoy Gateway", "Gateway API"]
---

In [my article on migrating to Envoy Gateway](/articles/ingress-nginx-controller-to-envoy-gateway/), I exposed Envoy through a `NodePort` service and pointed the router's port-forwards at the assigned ports, as a `NodePort` cannot use the standard HTTP and HTTPS ports. Accessing apps directly on the local LAN, however, then requires specifying a non-standard port in the URL. On a single-node homelab cluster, `hostPort` avoids this: it binds a container port directly on the node's network interface, so the node answers on ports 80 and 443 with nothing in between. Unlike `hostNetwork`, the pod keeps its own network namespace.

This article explains how to configure it with Envoy Gateway and the pitfalls to avoid.

## Configuration

Envoy Gateway creates the Envoy proxy `Deployment` itself, so the `hostPort` has to be added by customizing it through the `EnvoyProxy` resource:

```yaml
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: EnvoyProxy
metadata:
  name: eg
  namespace: envoy-gateway-system
spec:
  provider:
    type: Kubernetes
    kubernetes:
      envoyDeployment:
        strategy:
          type: Recreate
        patch:
          type: StrategicMerge
          value:
            spec:
              template:
                spec:
                  containers:
                    - name: envoy
                      ports:
                        - name: http-hostport
                          containerPort: 10080
                          hostPort: 80
                          protocol: TCP
                        - name: https-hostport
                          containerPort: 10443
                          hostPort: 443
                          protocol: TCP
```

The `EnvoyProxy` is attached to the `GatewayClass` through its `parametersRef`, as explained in the migration article. Two details are worth explaining.

### The container ports are offset by 10000

Envoy Gateway doesn't run Envoy as root, so it can't listen on ports below 1024. For these, it adds 10000 to the port of the `Gateway` listener: a listener on port 80 makes Envoy listen on 10080 inside the container, and 443 becomes 10443.

The `hostPort` is therefore what you want to expose (80 and 443), while the `containerPort` is the port Envoy really listens on (10080 and 10443). Setting `containerPort: 80` would map the node's port 80 to a port that nothing listens on.

### Use the `Recreate` strategy

The default `RollingUpdate` strategy starts the new pod before stopping the old one. On a single node, both pods can't hold the same `hostPort`, so the new pod stays `Pending` and the rollout never completes.

`Recreate` stops the old pod first. The price is a short interruption every time the proxy pod is restarted, such as when the `EnvoyProxy` is changed or Envoy Gateway is upgraded.

## Applying and verifying

Two things can't listen on the same port of a node, so ports 80 and 443 must be free before Envoy can use them. If anything else holds them, such as another ingress controller running with `hostNetwork: true`, it has to release them first.

Once the `EnvoyProxy` is applied, check that the pod really holds the ports:

```bash
kubectl get pods -n envoy-gateway-system -o jsonpath='{.items[*].spec.containers[?(@.name=="envoy")].ports}'
```

and that the `Gateway` answers on the standard ports, for example with `curl -I https://homepage.example.com`.

## Limitations

- **The CNI must support `hostPort`.** It is implemented by the CNI's portmap plugin. My cluster runs Talos Linux with Flannel, where it works out of the box.
- **The pod is tied to a node.** With `hostPort`, the node's IP address is the entry point of the cluster, and the proxy has to run on that node. This is fine with a single node. With several, a `LoadBalancer` service backed by something like MetalLB is more suitable, and rolling updates then become possible again.
- **Restarts cause downtime**, as mentioned above.

For a single-node homelab, I find these acceptable in exchange for being able to reach apps on the LAN without a port number.
