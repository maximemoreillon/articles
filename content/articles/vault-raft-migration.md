---
date: "2026-09-26T00:00:00+09:00"
title: "Migrating HashiCorp Vault from file storage to Raft"
tags: ["Kubernetes", "Homelab", "HashiCorp Vault"]
---

In [my article on ESO with HashiCorp Vault](/articles/vault-eso/), Vault was installed with the Helm chart's default configuration: a single server using **file storage** on a `PersistentVolumeClaim`. This works well, but backing it up is not trivial. The only option is to copy the files of the volume while Vault is running, for instance with [Velero](/articles/velero/), and nothing guarantees that such a copy is consistent, since a single write to Vault can touch several files.

Vault's **integrated storage**, based on the Raft consensus algorithm, solves this. Although it is designed for clusters of several Vault servers, it works just as well with a single one, and it is the storage HashiCorp recommends by default. Its main advantage here is that it can produce a consistent snapshot through the API, which is still encrypted by Vault and can therefore safely be stored off-site:

```bash
vault operator raft snapshot save backup.snap
```

This article explains how I migrated an existing single-server Vault from file storage to Raft.

## Helm values

With the Helm chart, Raft storage is enabled through the HA settings. The chart's default Raft configuration stores its data in `/vault/data`, just like the file storage, so only the following block has to be added to the values:

```yaml
server:
  ha:
    enabled: true
    replicas: 1
    disruptionBudget:
      enabled: false
    raft:
      enabled: true
      setNodeId: true
```

Two settings deserve an explanation:

- **`disruptionBudget`**: in HA mode, the chart creates a `PodDisruptionBudget` whose `maxUnavailable` is computed from the number of replicas. With a single replica, it is 0, which would prevent the node from ever being drained, for example during an OS upgrade.
- **`setNodeId`**: this sets the Raft node ID to the name of the pod, `vault-0`. The migration below has to use the same ID.

The `StatefulSet`, its `PersistentVolumeClaim` and the `vault` service all remain the same, so the migration happens on the same volume, and anything using `http://vault.vault:8200`, such as ESO, keeps working. The chart simply adds the `vault-active`, `vault-standby` and `vault-internal` services.

## Migration

Vault provides the `vault operator migrate` command to copy data from one storage to another. It works directly on the storage, so it requires no token, but Vault must be stopped while it runs. Before starting, make sure the unseal keys are at hand, because Vault will come back sealed.

### Backing up the data

As a safety net, the content of the volume can first be copied locally:

```bash
kubectl -n vault exec vault-0 -- tar czf - -C /vault data > vault-backup.tgz
```

This data is encrypted by Vault, so it is useless without the unseal keys.

### Stopping Vault

If Vault is deployed with ArgoCD, automated sync should be disabled for this step, as ArgoCD would otherwise scale it back up.

```bash
kubectl -n vault scale statefulset vault --replicas=0
```

### Running the migration

The migration runs in a one-off pod that mounts Vault's volume. It uses the same image as Vault, and the same user and group, so that the new files belong to the `vault` user:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: vault-migrate
  namespace: vault
spec:
  restartPolicy: Never
  securityContext:
    runAsNonRoot: true
    runAsUser: 100
    runAsGroup: 1000
    fsGroup: 1000
  containers:
    - name: migrate
      image: hashicorp/vault:2.0.2
      command:
        - /bin/sh
        - -c
        - |
          cat > /tmp/migrate.hcl <<'EOF'
          storage_source "file" {
            path = "/vault/data"
          }
          storage_destination "raft" {
            path    = "/vault/data"
            node_id = "vault-0"
          }
          cluster_addr = "https://vault-0.vault-internal:8201"
          EOF
          vault operator migrate -config=/tmp/migrate.hcl
      volumeMounts:
        - name: data
          mountPath: /vault/data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: data-vault-0
```

The `node_id` matches the one set by `setNodeId`, and `cluster_addr` matches the address the chart gives to the pod.

The logs of the pod list every key copied and should end with:

```
Success! All of the keys have been migrated.
```

Raft stores its data in a `vault.db` file and a `raft` folder. These sit next to the folders of the file storage (`auth`, `core`, `logical` and `sys`), which the migration leaves untouched:

```
$ ls -la /vault/data
drwx------    3 vault    vault           50 Sep 12 02:12 auth
drwx------    6 vault    vault         4096 Sep 26 01:09 core
drwx------    4 vault    vault           94 Sep 12 01:58 logical
drwx------    3 vault    vault           38 Sep 26 01:09 raft
drwx------    6 vault    vault           64 Sep 26 01:08 sys
-rw-------    1 vault    vault     16801792 Sep 26 01:09 vault.db
```

The pod can then be deleted.

### Switching to Raft

Once the Helm values are updated and applied, Vault starts using Raft. It then has to be unsealed with the same keys as before:

```bash
kubectl -n vault exec -it vault-0 -- vault operator unseal
```

## Verification

`vault status` should now report Raft as the storage type:

```
$ kubectl -n vault exec vault-0 -- vault status
Sealed                  false
Storage Type            raft
HA Enabled              true
HA Mode                 active
Raft Committed Index    112
Raft Applied Index      112
```

In my case, Vault was down for about three minutes. Since my `ExternalSecret` objects use `creationPolicy: Orphan`, the Kubernetes secrets they manage were not affected in the meantime. ESO only reported errors until Vault was available again, after which all secrets were synced successfully.

## Rollback and cleanup

As long as the file storage is still on the volume, rolling back only requires reverting the Helm values. Vault would then start on the file storage again, losing anything written since the migration.

Once Raft has proven reliable, the `auth`, `core`, `logical` and `sys` folders can be deleted from `/vault/data`, keeping `vault.db` and the `raft` folder.

## Next step: snapshots

Automated snapshots are a feature of Vault Enterprise, so with the free version, a `CronJob` is needed to call `vault operator raft snapshot save` regularly and upload the result to object storage, ideally on another site. Since snapshots are encrypted, restoring one requires the unseal keys, which should therefore be kept somewhere independent of the cluster.
