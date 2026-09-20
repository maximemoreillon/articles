---
date: "2026-09-20T00:00:00+09:00"
title: "Nextcloud Notes re-syncing everything behind Envoy Gateway"
tags: ["Kubernetes", "Homelab", "Envoy Gateway", "Nextcloud"]
---

After [moving from Ingress NGINX to Envoy Gateway](/articles/ingress-nginx-controller-to-envoy-gateway/), the Nextcloud Notes app on my phone started taking around 40 seconds to sync, with the loader spinning the whole time.

## The problem

The Envoy access log showed why: every sync downloaded the whole collection.

```
GET /index.php/apps/notes/api/v1/notes?pruneBefore=0   200   4.8 MB
```

With around 5,000 notes, that is a lot of data to fetch for a sync that usually has nothing to update. According to the [Notes API documentation](https://github.com/nextcloud/notes/blob/main/docs/api/v1.md), a client is supposed to send the `Last-Modified` timestamp and the `ETag` of the previous response, so that the server only returns what changed. Behind Ingress NGINX, this worked. Behind Envoy Gateway, `pruneBefore` was always `0`.

## The cause

Three things combine:

1. **Envoy lowercases HTTP/1.1 header names by default.** Ingress NGINX and Apache keep them as sent. The same file, requested over HTTP/1.1, returns:

   ```
   # Ingress NGINX / Apache
   Last-Modified: Tue, 11 Aug 2026 00:23:30 GMT
   ETag: "32f-658ba7af4d4b7"

   # Envoy Gateway
   last-modified: Tue, 11 Aug 2026 00:23:30 GMT
   etag: "32f-658ba7af4d4b7"
   ```

   This is compliant, since header names are case-insensitive ([RFC 9110](https://www.rfc-editor.org/rfc/rfc9110#name-field-names)).

2. **The Android client treats them as case-sensitive.** The Notes app reaches the server through the Nextcloud Files app and the Single Sign-On library. The Files app forwards header names exactly as received, and the library stores them in a plain [`HashMap`](https://github.com/nextcloud/Android-SingleSignOn/blob/main/lib/src/main/java/com/nextcloud/android/sso/api/ParsedResponse.java).

3. **The Notes app looks up `"ETag"` and `"Last-Modified"` exactly.** If they are not found, it [stores a modified time of 0 and no ETag](https://github.com/nextcloud/notes-android/blob/main/app/src/main/java/it/niedermann/owncloud/notes/persistence/NotesServerSyncTask.java), and the next request is `pruneBefore=0` again.

## The fix

Envoy Gateway can keep the original casing with a `ClientTrafficPolicy` on the `Gateway`:

```yaml
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: ClientTrafficPolicy
metadata:
  name: preserve-header-case
  namespace: envoy-gateway-system
spec:
  targetRefs:
    - group: gateway.networking.k8s.io
      kind: Gateway
      name: eg
  http1:
    preserveHeaderCase: true
```

After this, the headers keep the case sent by the backend, and syncs become incremental:

```
GET /index.php/apps/notes/api/v1/notes?pruneBefore=1789885232   200   69 KB
GET /index.php/apps/notes/api/v1/notes?pruneBefore=1789885288   304   0 B
```

A sync now takes under a second. The policy applies to all HTTP/1.1 traffic going through the `Gateway`, which is harmless for compliant clients. HTTP/2 is unaffected, as it always uses lowercase names.
