---
date: "2025-05-04T00:00:00+09:00"
title: "Backups with Velero"
tags: ["Kubernetes"]
---

The persistent volumes of my Talos cluster are provisioned by democratic-csi in TrueNAS, and TrueNAS uses replication to copy backups to another system. The Kubernetes manifests used to deploy applications in the cluster are also stored in a git repository. This gives a first disaster recovery path, but a proper backup strategy should rely on more than one mechanism.

As another layer, I installed [Velero](https://velero.io/), a popular backup tool for Cloud Native infrastructures. Velero backs up Kubernetes objects such as deployments and services, as well as the contents of persistent volumes, using Kopia.

## Installation

Although Velero can be installed using its own CLI, I used its Helm chart instead, as I don't like unnecessarily increasing the number of tools I need to manage my infrastructure.

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

`linode-object-storage-credentials` is a plain `Opaque` secret containing a `cloud` key in the standard AWS credentials file format:

```ini
[default]
aws_access_key_id=<access key>
aws_secret_access_key=<secret key>
```

## Why this setup

Between democratic-csi's replication on TrueNAS, the git repository holding my manifests, and Velero backing up both Kubernetes objects and volume contents to Linode Object Storage, I now have backups that don't all depend on a single failure domain.
