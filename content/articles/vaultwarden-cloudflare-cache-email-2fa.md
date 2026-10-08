---
date: '2026-10-08T08:00:05+09:00'
draft: true
title: "Vaultwarden hiding Email 2FA behind a Cloudflare tunnel"
tags: ["Kubernetes", "Homelab", "Vaultwarden", "Cloudflare"]
---

I run a [Vaultwarden](https://github.com/dani-garcia/vaultwarden) instance that can be reached through two hostnames: one goes through a Cloudflare tunnel, the other through a router port forward straight to Envoy Gateway. After configuring SMTP so that users could enable two-step login by email, the Email option appeared in the web vault's two-step login settings (`/#/settings/security/two-factor`) on the second hostname, but not on the first.

## The problem

Both hostnames lead to the same pod, so the difference had to come from what sits in front of it. The API returned the same server configuration either way:

```
curl -s https://vaultwarden.example.com/api/config
```

The web vault's `index.html` and its hashed JavaScript bundles were identical too. One file was not: `css/vaultwarden.css`.

## Vaultwarden hides features with CSS

The web vault is the regular Bitwarden web client, with a few patches. To hide what a Vaultwarden instance doesn't support or has disabled, Vaultwarden serves a stylesheet, `/css/vaultwarden.css`, that it generates from a template and its current configuration. When no SMTP server is configured, the generated file hides the Email provider:

```css
.providers-2fa-1 { display: none !important }
```

The signup link is hidden in the same way when signups are disabled. Unlike the rest of the web vault, this file has no hash in its name, so its URL stays the same when its content changes. Vaultwarden serves it with `cache-control: public, no-cache`, which tells caches to check with the server before reusing it.

## The cause

Comparing the response headers through both hostnames showed the issue:

```
# Through the port forward (served by Vaultwarden)
cache-control: public, no-cache

# Through the Cloudflare tunnel
cache-control: public, max-age=86400
cf-cache-status: HIT
age: 35967
last-modified: Wed, 07 Oct 2026 12:49:22 GMT
```

Cloudflare was serving a copy of the stylesheet it had cached about ten hours earlier, when the first pod had just started and SMTP was not configured yet. The two files differed in exactly two ways: the cached copy hid the Email provider and still showed the signup link, which matched the configuration at the time.

Cloudflare caches files with static extensions such as `.css` by default. Here, the cached copy came with `max-age=86400`, so it would have stayed in place for a day, and browsers were told to keep it for a day as well.

## The fix

Purging the URL in the Cloudflare dashboard (Caching → Configuration → Custom Purge) brought the Email option back after a hard refresh of the page.

To keep this from happening again after the next configuration change, caching can be turned off for the whole hostname with a Cache Rule (Caching → Cache Rules → Create rule):

- **If incoming requests match**: `Hostname` equals `vaultwarden.example.com`
- **Cache eligibility**: Bypass cache
- **Browser TTL**, if offered: respect the origin's headers

For a password manager, caching at the edge gains almost nothing, as the web vault is small and the clients mostly talk to the API. After deploying the rule, the stylesheet should come back with Vaultwarden's own header:

```
curl -sI https://vaultwarden.example.com/css/vaultwarden.css | grep -iE 'cf-cache-status|cache-control'
```

```
cache-control: public, no-cache
cf-cache-status: DYNAMIC
```

If the response still shows `max-age=86400`, also check the zone's Browser Cache TTL setting (Caching → Configuration), which overrides the origin's header when set to a fixed value.
