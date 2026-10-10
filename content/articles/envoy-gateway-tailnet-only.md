---
date: '2026-10-10T12:31:20+09:00'
draft: true
title: "Tailnet-only services with Envoy Gateway, Cloudflare DNS and Tailscale"
tags: ["Kubernetes", "Homelab", "Envoy Gateway", "Cloudflare", "Tailscale"]
---

Some services in my Kubernetes cluster, like HashiCorp Vault, should only be reachable from devices on my [Tailscale](https://tailscale.com/) tailnet. They used to be exposed through NodePorts on the node's Tailscale IP, which works but means remembering port numbers. This article shows how to give them proper hostnames through [Envoy Gateway](https://gateway.envoyproxy.io/) while keeping them off the internet and the LAN.

Three pieces are involved:

- **Cloudflare DNS**: a public record pointing at the node's Tailscale IP
- **Tailscale**: split DNS, so devices can resolve that record wherever they are
- **Envoy Gateway**: a dedicated listener, restricted by client IP

## Cloudflare: a DNS-only record

In the Cloudflare dashboard, create an `A` record for a wildcard under a dedicated subdomain, pointing at the Tailscale IP of the Kubernetes node:

- **Name**: `*.k8s.tailnet.example.com`
- **IPv4 address**: `100.x.y.z`
- **Proxy status**: DNS only (grey cloud)

The record has to be DNS only. A proxied record would make Cloudflare connect to `100.x.y.z` itself, which it cannot reach. Since the record is not proxied, Cloudflare settings such as "Always Use HTTPS" don't apply to it either.

The record is public, so anyone can look it up, but this isn't a problem: `100.64.0.0/10` addresses are only reachable by devices on the tailnet. The extra `k8s` label leaves room for other machines (`*.nas.tailnet.example.com`, etc.).

## Tailscale: split DNS

Many resolvers, including dnsmasq and Unbound on home routers, have DNS rebinding protection. They drop answers from public DNS that point to private ranges, and `100.64.0.0/10` counts as one. Through such a resolver, the lookup returns `NOERROR` with no address:

```
dig vault.k8s.tailnet.example.com @192.168.1.1 | grep -E 'status|ANSWER'
```

```
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 12153
;; flags: qr rd ra; QUERY: 1, ANSWER: 0, AUTHORITY: 0, ADDITIONAL: 1
```

The router could be configured to allow it, but that would only fix one network. Instead, in the Tailscale admin console under DNS → Nameservers, add a custom nameserver:

- **Nameserver**: `1.1.1.1` (and `1.0.0.1`)
- **Restrict to domain**: `tailnet.example.com`

Every device that accepts the tailnet's DNS settings now sends lookups for `*.tailnet.example.com` to Cloudflare's resolver through Tailscale's local resolver (`100.100.100.100`), whatever network it is on. Other lookups still go to the local resolver. The result can be checked with:

```
tailscale dns status
resolvectl query vault.k8s.tailnet.example.com
```

## Envoy Gateway: a restricted listener

On the Envoy side, a listener dedicated to the tailnet hostnames is added to the `Gateway`. It can share a port with other listeners: they are told apart by hostname, and the more specific one wins:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: eg
  namespace: envoy-gateway-system
spec:
  gatewayClassName: eg
  listeners:
    # Tailnet-only hosts, restricted by tailnet-allowlist
    - name: tailnet
      protocol: HTTP
      port: 80
      hostname: "*.k8s.tailnet.example.com"
      allowedRoutes:
        namespaces:
          from: All
```

Plain HTTP is acceptable here because Tailscale traffic is already encrypted by WireGuard. On a single node, binding Envoy to host ports 80/443 (with a `Recreate` deployment strategy, since a rolling update can't start a second pod on the same host ports) means URLs need no port number.

The hostname alone doesn't protect anything: a client on the internet or the LAN could send `Host: vault.k8s.tailnet.example.com` to the node directly. So a `SecurityPolicy` denies every request on that listener unless it comes from an allowed address:

```yaml
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: SecurityPolicy
metadata:
  name: tailnet-allowlist
  namespace: envoy-gateway-system
spec:
  targetRefs:
    - group: gateway.networking.k8s.io
      kind: Gateway
      name: eg
      sectionName: tailnet
  authorization:
    defaultAction: Deny
    rules:
      - name: allow-tailnet
        action: Allow
        principal:
          clientCIDRs:
            - 10.244.0.1/32 # Tailscale clients, masqueraded (see below)
```

Finally, routes opt in with `sectionName`. The Vault Helm chart can create the route itself:

```yaml
server:
  httproute:
    enabled: true
    hostnames:
      - vault.k8s.tailnet.example.com
    parentRefs:
      - name: eg
        namespace: envoy-gateway-system
        sectionName: tailnet
```

Routes for other sites carry their own hostnames, which don't overlap with `*.k8s.tailnet.example.com`, so they don't attach to the new listener.

### Which source address does Envoy see?

The allowlist only works if it matches the address Envoy actually sees, which depends on how traffic reaches the pod. Envoy's access log shows it in `downstream_remote_address`, so it is worth sending a request from each kind of client and checking:

```
kubectl -n envoy-gateway-system logs deploy/<envoy-deployment> -c envoy --since=1m \
  | grep -o '"downstream_remote_address":"[^"]*"\|":authority":"[^"]*"' | paste - -
```

In my case, with Envoy bound to host ports 80/443 and also exposed through a NodePort with `externalTrafficPolicy: Local`:

| Client | Address seen by Envoy |
| --- | --- |
| Tailscale, through the node's tailnet IP | `10.244.0.1`, the CNI bridge gateway |
| Internet, through a router port forward | the client's real public IP |

Tailscale traffic was masqueraded the same way through the host port and the NodePort, so the original `100.x` address could not be matched, but the bridge address could. Public traffic doesn't match it, which is what matters. Nothing else should be allowed: in particular, the pod network must stay out of the list, or anything that proxies requests from inside the cluster would be let in.

A request from inside the cluster checks that the policy is enforced, since a pod's IP is not on the list:

```
kubectl -n envoy-gateway-system run probe --rm -i --restart=Never --image=curlimages/curl \
  --command -- curl -s -o /dev/null -w '%{http_code}\n' \
  -H 'Host: vault.k8s.tailnet.example.com' http://<envoy-service>/
```

```
403
```

From a device on the tailnet, `http://vault.k8s.tailnet.example.com` now reaches Vault, while every other source gets a 403.
