---
date: "2026-09-17T00:00:00+09:00"
title: "Migrating from Ingress NGINX Controller to Envoy Gateway"
tags: ["Kubernetes", "Homelab", "Envoy Gateway", "Gateway API"]
---

As the [Ingress NGINX controller](https://github.com/kubernetes/ingress-nginx) is now deprecated, replacing it with the Gateway API is in order. The Gateway API is a newer Kubernetes standard for traffic routing with several available implementations — [Envoy Gateway](https://gateway.envoyproxy.io/), Istio, Cilium, and NGINX Gateway Fabric among them. Here, I have chosen Envoy Gateway as it is backed by the CNCF and provides all needed features, including native OIDC authentication, which removes the need for Oauth2-Proxy (see [this article](/articles/retiring-oauth2-proxy/)).

This article explains my migration process: preparing TLS certificates, setting up Envoy Gateway, converting the `Ingress` of an application into an `HTTPRoute`, and switching traffic over.

## Preparing the terrain: TLS certificates

I use cert-manager to provision certificates for my applications. With `Ingress` objects, this is self-contained: each app has its own ingress, whose `spec.tls` block maps its hostname to a certificate, and an annotation tells cert-manager's ingress-shim which issuer to use. The common `HTTP-01` challenge therefore works well, and adding a new app doesn't require changing any shared resource.

Envoy Gateway is different because cert-manager's equivalent integration, gateway-shim, operates at the `Gateway` listener level rather than per `HTTPRoute`. Using `HTTP-01` there would require a dedicated listener in the shared `Gateway` resource for every hostname, each with its own hostname and certificate reference. Every new app would then mean editing that shared file, instead of just adding its own self-contained route.

To avoid this, I use a single wildcard certificate that one HTTPS listener can reference for all apps:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: wildcard
  namespace: envoy-gateway-system
spec:
  secretName: wildcard-tls
  issuerRef:
    name: letsencrypt-dns
    kind: ClusterIssuer
  dnsNames:
    - "*.example.com"
```

Wildcard certificates can only be validated with the `DNS-01` challenge, which is what the `letsencrypt-dns` issuer is set up for. See cert-manager's documentation on [configuring DNS01 challenge providers](https://cert-manager.io/docs/configuration/acme/dns01/) for how to set one up.

## Envoy Gateway install and setup

As per [the official documentation](https://gateway.envoyproxy.io/docs/install/install-helm/), Envoy Gateway can be installed using Helm.

Since my setup is a homelab running as a single-node Kubernetes cluster, there is no cloud load balancer to provision a `LoadBalancer` service. Instead, I'm configuring Envoy Gateway to receive traffic through a `NodePort`, to which the router forwards incoming traffic. This is achieved by creating an `EnvoyProxy` object (CRD) as follows:

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
      envoyService:
        type: NodePort
```

This `EnvoyProxy` is then used to create the following `GatewayClass`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: eg
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
  parametersRef:
    group: gateway.envoyproxy.io
    kind: EnvoyProxy
    name: eg # Must match EnvoyProxy created above
    namespace: envoy-gateway-system
```

The `GatewayClass` can then be used to instantiate a `Gateway`. Here, the wildcard certificate created above is used to terminate TLS on the corresponding listener.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: eg
  namespace: envoy-gateway-system
spec:
  gatewayClassName: eg # Must match GatewayClass created above
  listeners:
    - name: http
      protocol: HTTP
      port: 80
      allowedRoutes:
        namespaces:
          from: All
    - name: https
      protocol: HTTPS
      port: 443
      tls:
        certificateRefs:
          - name: wildcard-tls
      allowedRoutes:
        namespaces:
          from: All
```

## Converting Ingresses to HTTPRoutes

Each application that was exposed through an `Ingress` needs an `HTTPRoute` instead. Here is the `Ingress` of Homepage, for reference:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: homepage
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt
spec:
  tls:
    - hosts:
        - homepage.example.com
      secretName: homepage-tls
  rules:
    - host: homepage.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: homepage
                port:
                  number: 3000
```

The `letsencrypt` issuer referenced in the annotation is the old `HTTP-01` issuer discussed in the section on TLS certificates, not the `letsencrypt-dns` issuer used for the wildcard certificate.

With the `Gateway` available, the equivalent `HTTPRoute` looks as follows:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: homepage
spec:
  hostnames:
    - homepage.example.com
  parentRefs:
    - name: eg
      namespace: envoy-gateway-system
  rules:
    - backendRefs:
        - name: homepage
          port: 3000
```

Note that the `HTTPRoute` has no TLS configuration: the `tls` block and the cert-manager annotation are gone, as the wildcard certificate on the `Gateway` listener already terminates TLS for every hostname.

Once the `HTTPRoutes` are in place, incoming traffic can be switched from the Ingress NGINX controller to Envoy Gateway by updating the router's port-forwards or the load balancer configuration so that they point to the NodePorts assigned to Envoy Gateway instead of the ports used by Ingress NGINX.
