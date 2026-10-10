---
date: "2026-10-11T12:00:00+09:00"
title: "Envoy Gateway HTTPS redirect"
tags: ["Kubernetes", "Homelab", "Envoy Gateway"]
---

The following is a manifest to create an HTTPS redirection for `HTTPRoute` objects. In this example, all `HTTProute`s subdomains of `example.com` are handled by both an HTTP and HTTPS listener, named `http` and `https` respectively.

```yml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: https-redirect
  namespace: envoy-gateway-system
spec:
  hostnames:
    - "*.example.com"
  parentRefs:
    - group: gateway.networking.k8s.io
      kind: Gateway
      name: eg
      sectionName: http
  rules:
    - filters:
        - type: RequestRedirect
          requestRedirect:
            scheme: https
            statusCode: 308
      matches:
        - path:
            type: PathPrefix
            value: /
```
