---
date: "2025-03-10"
title: "Dynamic storage provisioning with democratic-csi"
tags: ["Kubernetes", "TrueNAS"]
---

My Talos cluster needed persistent storage backed by the TrueNAS VM that already manages my disks with ZFS. Rather than provisioning volumes on TrueNAS by hand for every workload, I installed [democratic-csi](https://github.com/democratic-csi/democratic-csi), a CSI driver that lets Kubernetes provision storage on TrueNAS directly.

## How it works

democratic-csi talks to the TrueNAS API. When a `PersistentVolumeClaim` is created in the cluster, it creates a matching resource on TrueNAS and hands it back to Kubernetes as a `PersistentVolume`, so provisioning stays entirely declarative — no manual dataset or zvol creation.

I ended up with two `StorageClass`es on the cluster, covering the two access modes I need:

- **NFS**, backed by ZFS datasets, for volumes that need `ReadWriteMany`.
- **iSCSI**, backed by ZFS zvols, for block storage with `ReadWriteOnce`.

## Why this setup

Letting Kubernetes drive storage provisioning through the TrueNAS API keeps volume lifecycle entirely declarative and tied to the cluster's manifests, while ZFS underneath still gives me snapshots, compression and scrubbing for free. It's also what makes [replicating the pool to my secondary server in Switzerland](/articles/secondary-server-switzerland/) straightforward — it builds directly on top of it.
