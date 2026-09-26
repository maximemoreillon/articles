---
date: "2026-09-26T00:00:00+09:00"
title: "Off-site Vault backups with Raft snapshots and a CronJob"
tags: ["Kubernetes", "Homelab", "HashiCorp Vault"]
---

In [my previous article](/articles/vault-raft-migration/), I migrated HashiCorp Vault from file storage to Raft, mainly so that it could be backed up properly: with Raft, `vault operator raft snapshot save` produces a consistent snapshot of the whole Vault in a single file. Vault Enterprise can take such snapshots automatically, but with the free version, they have to be scheduled some other way.

This article explains how I use a Kubernetes `CronJob` to take a daily snapshot of Vault and upload it to S3-compatible object storage located on another site, in my case a [RustFS](https://rustfs.com/) instance running in another cluster.

## Overview

The `CronJob` runs a pod with two containers sharing an `emptyDir` volume:

1. An init container, using the Vault image, logs in to Vault with its Kubernetes service account and saves a snapshot into the shared volume.
2. The main container, using the [rclone](https://rclone.org/) image, uploads the snapshot to the bucket and deletes the snapshots older than 30 days.

Snapshots are encrypted by Vault, so they can safely be stored off-site. Restoring one, however, requires the unseal keys, which should therefore be kept somewhere independent of the cluster.

## Vault configuration

### ACL policy

Taking a snapshot requires read access to a single path. In Access control, create an ACL policy named `raft-snapshot`:

```hcl
path "sys/storage/raft/snapshot" {
  capabilities = ["read"]
}
```

### Kubernetes auth role

The job logs in using the Kubernetes authentication method set up in [my article on ESO](/articles/vault-eso/). A dedicated role restricts this login to the service account of the job, and gives it the policy above only. It can be created with the CLI of the Vault web UI:

```
write auth/kubernetes/role/vault-snapshot bound_service_account_names=vault-snapshot bound_service_account_namespaces=vault-snapshot token_policies=raft-snapshot token_ttl=10m
```

Tokens obtained with this role are only valid for 10 minutes, which is more than enough for the job.

### S3 credentials

On the object storage side, create a bucket, here `vault-snapshots`, and an access key that can only access it. The key is then stored in Vault, in the `k8s` secret engine, as a secret named `vault-snapshots` with the fields `ACCESS_KEY_ID` and `SECRET_ACCESS_KEY`. ESO turns it into a Kubernetes secret for the job.

## Manifests

All resources live in a dedicated `vault-snapshot` namespace. Here, they are deployed by an ArgoCD application with the `CreateNamespace=true` sync option.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: vault-snapshot
  namespace: vault-snapshot
---
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: s3
  namespace: vault-snapshot
spec:
  refreshInterval: 1h0m0s
  secretStoreRef:
    name: vault
    kind: ClusterSecretStore
  target:
    name: s3
    creationPolicy: Orphan
  dataFrom:
    - extract:
        key: vault-snapshots
---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: vault-snapshot
  namespace: vault-snapshot
spec:
  schedule: "0 3 * * *"
  timeZone: Asia/Tokyo
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 1
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      backoffLimit: 1
      template:
        spec:
          serviceAccountName: vault-snapshot
          restartPolicy: Never
          securityContext:
            runAsNonRoot: true
            runAsUser: 100
            runAsGroup: 1000
          initContainers:
            - name: snapshot
              image: hashicorp/vault:2.0.2
              env:
                - name: VAULT_ADDR
                  value: http://vault-active.vault:8200
              command:
                - /bin/sh
                - -c
                - |
                  set -e
                  export VAULT_TOKEN=$(vault write -field=token auth/kubernetes/login role=vault-snapshot \
                    jwt=@/var/run/secrets/kubernetes.io/serviceaccount/token)
                  vault operator raft snapshot save /snapshots/vault-$(date +%Y%m%d-%H%M%S).snap
                  ls -l /snapshots
              volumeMounts:
                - name: snapshots
                  mountPath: /snapshots
          containers:
            - name: upload
              image: rclone/rclone:1.75.1
              env:
                - name: RCLONE_CONFIG_DEST_TYPE
                  value: s3
                - name: RCLONE_CONFIG_DEST_PROVIDER
                  value: Other
                - name: RCLONE_CONFIG_DEST_ENDPOINT
                  value: https://s3.example.com
                - name: RCLONE_CONFIG_DEST_ACCESS_KEY_ID
                  valueFrom:
                    secretKeyRef:
                      name: s3
                      key: ACCESS_KEY_ID
                - name: RCLONE_CONFIG_DEST_SECRET_ACCESS_KEY
                  valueFrom:
                    secretKeyRef:
                      name: s3
                      key: SECRET_ACCESS_KEY
              command:
                - /bin/sh
                - -c
                - |
                  set -e
                  rclone copy /snapshots dest:vault-snapshots/my-cluster/ -v
                  rclone delete dest:vault-snapshots/my-cluster/ --min-age 30d -v
              volumeMounts:
                - name: snapshots
                  mountPath: /snapshots
          volumes:
            - name: snapshots
              emptyDir: {}
```

A few details are worth explaining:

- **`vault write auth/kubernetes/login`** exchanges the token of the pod's service account for a Vault token. The `@` prefix tells the Vault CLI to read the value from a file.
- **`vault-active`** is one of the services the Helm chart creates in Raft mode. It always points to the active Vault server, which is the one able to take a snapshot.
- **rclone is configured entirely through environment variables**: variables named `RCLONE_CONFIG_<REMOTE>_<OPTION>` define a remote, here named `dest`, so no configuration file is needed. As a result, rclone prints a harmless notice that its configuration file was not found.
- **Retention** is handled by rclone itself, with `rclone delete --min-age 30d`, so it doesn't depend on the object storage supporting lifecycle rules.
- **`concurrencyPolicy: Forbid`** prevents a new job from starting while the previous one is still running.

## Testing

Rather than waiting for the schedule, a job can be created from the `CronJob` right away:

```bash
kubectl -n vault-snapshot create job --from=cronjob/vault-snapshot snapshot-test
```

The logs of the `snapshot` container show the snapshot that was saved:

```
$ kubectl -n vault-snapshot logs job/snapshot-test -c snapshot
total 40
-rw-------    1 vault    vault        38101 Sep 26 01:29 vault-20260926-012912.snap
```

and those of the `upload` container confirm that it reached the bucket:

```
$ kubectl -n vault-snapshot logs job/snapshot-test -c upload
INFO  : vault-20260926-012912.snap: Copied (new)
Transferred:   	   37.208 KiB / 37.208 KiB, 100%, 0 B/s, ETA -
Transferred:            1 / 1, 100%
Elapsed time:         1.6s
```

The test job can then be deleted.

## Restoring

A snapshot is restored into a Raft-based Vault that is initialized and unsealed:

```bash
vault operator raft snapshot restore -force vault-20260926-012912.snap
```

The `-force` flag is required when the snapshot comes from a different Vault, such as a fresh one set up after losing the original. After the restore, the Vault uses the keys of the snapshot, so it has to be unsealed with the original unseal keys. A backup that has never been restored is only a hope, so this is worth testing once, for example with a throwaway Vault.
