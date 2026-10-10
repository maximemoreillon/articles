---
date: "2026-10-11T12:00:00+09:00"
title: 'Fixing Keycloak''s "HTTPS required" behind cloudflared and Envoy Gateway'
tags: ["Kubernetes", "Homelab", "Envoy Gateway", "Cloudflare", "Keycloak"]
---

In my Kubernetes cluster, public services are exposed through a [Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/): `cloudflared` runs in the cluster and forwards requests to [Envoy Gateway](https://gateway.envoyproxy.io/), which routes them to the applications. When I published [Keycloak](https://www.keycloak.org/) this way, every page answered with a 403 and the message "HTTPS required", although the browser was on `https://`. This article explains why and shows a small fix at the route level.

## The setup

The request goes through three hops:

- **Browser → Cloudflare**: HTTPS, terminated by Cloudflare
- **Cloudflare → cloudflared**: through the tunnel
- **cloudflared → Envoy**: plain HTTP, to the `http` listener of the `Gateway`

In the Cloudflare dashboard, the tunnel's public hostname points at Envoy's service:

- **Hostname**: `keycloak.example.com`
- **Service**: `http://<envoy-service>.envoy-gateway-system.svc:80`

Keycloak runs with plain HTTP enabled and trusts the proxy headers:

```yaml
env:
  - name: KC_HTTP_ENABLED
    value: "true"
  - name: KC_PROXY_HEADERS
    value: xforwarded
```

## Why Keycloak refuses the request

With `KC_PROXY_HEADERS=xforwarded`, Keycloak takes the scheme from `X-Forwarded-Proto` and the client address from `X-Forwarded-For`. The `master` realm requires SSL for external requests by default (`sslRequired: external`), and a visitor coming from the internet is external.

Cloudflare does send `X-Forwarded-Proto: https`. Envoy, however, doesn't trust that header from its downstream and sets it to the scheme of the connection it actually received. cloudflared connects over plain HTTP, so Keycloak receives `X-Forwarded-Proto: http`, concludes that an external client is using HTTP and answers "HTTPS required".

## Why not just disable "Require SSL"?

Each realm has a **Require SSL** setting (Realm settings → General), and setting it to "none" makes the error go away:

```
kcadm.sh update realms/master -s sslRequired=NONE
```

It only hides the symptom, though. Keycloak still believes the request came over HTTP, so it builds its URLs with `http://`: the issuer in the discovery document and in tokens, redirects, and form actions. OIDC clients expecting an `https://` issuer then reject it. It also removes a protection from the realm, which for `master` is the one that controls everything. Fixing the header instead lets Keycloak see the scheme the browser actually used.

## The fix: set the header on the route

A dedicated `HTTPRoute` for the tunnel hostname, attached only to the `http` listener, overwrites the header before the request reaches Keycloak:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: keycloak-cloudflare-tunnel
  namespace: keycloak
spec:
  hostnames:
    - keycloak.example.com
  parentRefs:
    - name: eg
      namespace: envoy-gateway-system
      sectionName: http
  rules:
    - filters:
        - type: RequestHeaderModifier
          requestHeaderModifier:
            set:
              - name: X-Forwarded-Proto
                value: https
      backendRefs:
        - name: keycloak
          port: 8080
      matches:
        - path:
            type: PathPrefix
            value: /
```

Forcing `https` is safe here because the hostname is only reachable through Cloudflare, which serves it over HTTPS only. The filter is scoped with `sectionName: http`, so it doesn't affect anything arriving on the `https` listener.

The discovery document shows whether Keycloak now sees the right scheme: its URLs should all start with `https://`.

```
curl -s https://keycloak.example.com/realms/master/.well-known/openid-configuration \
  | jq '{issuer, authorization_endpoint, token_endpoint}'
```

```json
{
  "issuer": "https://keycloak.example.com/realms/master",
  "authorization_endpoint": "https://keycloak.example.com/realms/master/protocol/openid-connect/auth",
  "token_endpoint": "https://keycloak.example.com/realms/master/protocol/openid-connect/token"
}
```

Two details matter:

- The hostname must not also be listed in another route attached to the `http` listener. When two routes match the same hostname and path, Envoy Gateway keeps the oldest one, and the header filter would silently be ignored.
- Other apps behind the tunnel can have the same problem in a different form, for example an OIDC client building an `http://` callback URL. Each one needs the same filter, or its external URL pinned to `https://`.

## The cleaner alternative

The workaround fixes one route at a time. A more general solution is to point cloudflared at Envoy's `https` listener instead (`https://<envoy-service>.envoy-gateway-system.svc:443`, with "Match SNI to Host" enabled), so Envoy sets `X-Forwarded-Proto: https` by itself. This requires Envoy to serve a valid certificate for the tunnel hostnames, and the tunnel then depends on that certificate being renewed. Until then, the route above keeps Keycloak working.
