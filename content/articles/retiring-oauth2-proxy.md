---
date: "2026-09-18T00:00:00+09:00"
title: "Retiring Oauth2-Proxy in favor of Envoy Gateway"
tags: ["Kubernetes", "Envoy Gateway", "Gateway API"]
---

Oauth2-Proxy is a convenient solution to add authentication with OIDC on applications that do not ship with it natively. It integrates well with the [Ingress NGINX controller](https://github.com/kubernetes/ingress-nginx) but the latter is now deprecated.

A replacement for Ingress NGINX is the Gateway API, a newer Kubernetes standard for traffic routing with several available implementations — [Envoy Gateway](https://gateway.envoyproxy.io/), Istio, Cilium, and NGINX Gateway Fabric among them. This article uses Envoy Gateway. Since Envoy Gateway provides its own OIDC authentication, Oauth2-Proxy is no longer needed for this purpose.

This article presents how to replace Oauth2-Proxy with Envoy Gateway's native OIDC authentication solution. For this purpose, the following `HTTPRoute` for the popular dashboard application [Homepage](https://gethomepage.dev/) is used as example:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: homepage
spec:
  hostnames:
    - homepage.example.com
  parentRefs:
    - group: gateway.networking.k8s.io
      kind: Gateway
      name: eg
      namespace: envoy-gateway-system
  rules:
    - backendRefs:
        - group: ""
          kind: Service
          name: homepage
          port: 3000
          weight: 1
      matches:
        - path:
            type: PathPrefix
            value: /
```

Enforcing OIDC authentication can be achieved by creating a `SecurityPolicy` object, here using Keycloak as IdP:

```yaml
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: SecurityPolicy
metadata:
  name: homepage-oidc
spec:
  targetRefs:
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      name: homepage
  oidc:
    provider:
      issuer: https://keycloak.example.com/realms/example
    clientID: envoy-gateway
    clientSecret:
      name: homepage-oidc-settings
    cookieDomain: example.com
```

Here, the OIDC client secret is fetched from a `Secret` object named `homepage-oidc-settings`. Note that the key must be literally `client-secret`.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: homepage-oidc-settings
type: Opaque
stringData:
  client-secret: "<CLIENT_SECRET>"
```

With the `SecurityPolicy` in place, accessing Homepage should now only be possible after authenticating towards the configured IdP.
