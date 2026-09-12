---
date: "2026-04-18"
title: "HashiCorp Vault OIDC with Keycloak"
tags: ["Kubernetes", "HashiCorp Vault", "Keycloak"]
---

[HashiCorp Vault](https://www.hashicorp.com/en/products/vault) supports authenticating users via OIDC, which means it can delegate login to an existing [Keycloak](https://www.keycloak.org/) instance instead of managing its own set of credentials.

## Keycloak client settings

In Keycloak, create a client for Vault with _Client authentication_ set to `ON`, so that Keycloak issues a client secret for it.

## Policy

Vault's OIDC role needs to be mapped to a policy. The built-in `root` policy cannot be used for this, so a separate `admin` policy granting the needed capabilities has to be created instead and used in the role below.

## Auth method configuration

In Vault, enable the `OIDC` auth method at a `keycloak` path, so that it matches the role command below:

```
vault auth enable -path=keycloak oidc
```

Then configure it with:

- OIDC discovery URL, pointing at the Keycloak realm
- OIDC client ID, from the Keycloak client created above
- OIDC client secret, from the same client

## Role

A role then ties the OIDC method to the policy created above:

```
vault write auth/keycloak/role/default \
  allowed_redirect_uris="https://vault.example.com/ui/vault/auth/keycloak/oidc/callback" \
  user_claim="sub" \
  policies="admin"
```

`allowed_redirect_uris` must exactly match the URL(s) Vault is actually reachable at, callback path included; multiple values can be passed if Vault is reachable through more than one URL (e.g. an internal hostname and a NodePort). It is not yet clear which of these values Vault actually checks against, so when in doubt include every URL through which the UI can be reached.

## Troubleshooting

`Authentication failed: Missing auth_url. Please check that allowed_redirect_uris for the role include this mount path.`

This means the URL used to reach Vault's UI isn't present in `allowed_redirect_uris` for the role — add it and rerun the `vault write` command above.
