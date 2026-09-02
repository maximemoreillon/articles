---
date: "2025-06-15"
title: "Proxmox homelab with Talos and TrueNAS"
tags: ["Homelab", "Proxmox", "Talos", "Kubernetes", "Projects"]
---

I recently rebuilt my homelab around [Proxmox](https://www.proxmox.com/) as the hypervisor and [Talos Linux](https://www.talos.dev/) for running applications. This is a short overview of how it is put together.

## Hardware

Almost all of it was bought second-hand, mostly on [Mercari](https://jp.mercari.com/):

- **Motherboard:** ASUS H270 Pro (ATX)
- **CPU:** Intel Core i7-7700
- **Memory:** 64 GB DDR4
- **Proxmox boot:** 2× Crucial MX500 256 GB SATA SSD
- **VM storage:** 2× Crucial P5 Plus NVMe SSD
- **HBA:** Inspur 9300-8i flashed to IT mode, passed through to the TrueNAS VM
- **PSU:** Seasonic Focus SSR-650FM (650 W, semi-modular)
- **Chassis:** Chieftec UNC-411E-B 4U, mounted in a StarTech 18U rack

Every pair of drives is set up as a ZFS mirror — the boot SSDs and the VM NVMe drives on Proxmox, and on the HBA, TrueNAS manages two more mirrored pools:

- 2× 2 TB Seagate IronWolf — HDD mirror
- 2× Crucial MX500 500 GB — SSD mirror

Buying used keeps the cost down, and the i7-7700 / H270 platform has more than enough headroom for what the lab actually does.

## Virtualisation

A single Proxmox host runs the whole setup as virtual machines:

- **TrueNAS** runs as a VM and handles storage. Disks are passed through to it so that ZFS manages them directly, and it exposes that storage back to the other VMs over NFS and iSCSI.
- **Talos Linux** provides a minimal, immutable, API-driven Kubernetes OS. All of my self-hosted applications run here rather than directly on the host.

Keeping everything as VMs on Proxmox makes it easy to snapshot, back up and rebuild individual pieces without touching the rest.

## Storage

Persistent volumes for the Talos cluster are provisioned dynamically on the TrueNAS VM using [democratic-csi](https://github.com/democratic-csi/democratic-csi). It talks to the TrueNAS API, so when a `PersistentVolumeClaim` is created in Kubernetes, democratic-csi creates a matching resource on TrueNAS:

- **Datasets** exported over NFS, for volumes that need `ReadWriteMany` access.
- **Zvols** exported over iSCSI, for block storage with `ReadWriteOnce` access.

The volume is then mounted into the pod like any other. This keeps all of the data on ZFS — with its snapshots, compression and scrubbing — while Kubernetes handles provisioning and lifecycle.

## Why this setup

Talos removes most of the maintenance that comes with running a general-purpose Linux distribution under Kubernetes: there is no SSH, no package manager and no config drift, just a declarative machine configuration. Combined with Proxmox for virtualisation and TrueNAS for storage, it gives me a homelab that is straightforward to reason about and quick to recover when something breaks.
