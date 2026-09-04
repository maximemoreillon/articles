---
date: "2025-05-04T00:00:00+09:00"
title: "Backups with Velero"
tags: ["Kubernetes"]
---

The persistent volumes of my Talos cluster are provisioned by democratic-csi in TrueNAS and the latter uses replication to take backups into another system. On the other hand the Kubernetes manifests nused to deploy applications in the cluster are stored in a git repository. This offers a first disaster recovery solution but a proper backup strategy should rely on more than one.

As another backup solution, I installed [Velero](https://velero.io/), a popular backup strategy for Cloud Native infrastructures.
Velero makes backups of K8s objects such as deployments or services as well PV content using Kopia.

Although Velero can be installed using its own CLI, I used its Helm chart instead as I don't like unnecessarily increase the number of tools I need to handle to manage my infrastructure.

In my case, the backup destination is Linode Object Storage.

The Helm values are as follows:

```yml
deployNodeAgent: true

initContainers:
  - name: velero-plugin-for-aws
    image: velero/velero-plugin-for-aws:v1.14.1
    volumeMounts:
      - mountPath: /target
        name: plugins

credentials:
  existingSecret: linode-object-storage-credentials

configuration:
  defaultVolumesToFsBackup: true

  backupStorageLocation:
    - name: default
      provider: aws
      bucket: velero
      config:
        region: jp-osa-1
        s3ForcePathStyle: "true"
        s3Url: https://jp-osa-1.linodeobjects.com
        checksumAlgorithm: ""

  volumeSnapshotLocation:
    - name: default
      provider: aws
      config:
        region: jp-osa-1
```
