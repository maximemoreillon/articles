---
date: "2025-06-15"
title: "Proxmox homelab with Talos and TrueNAS"
tags: ["Homelab", "Proxmox", "Talos", "Kubernetes", "Projects"]
---

I recently rebuilt my homelab around [Proxmox](https://www.proxmox.com/) as the hypervisor and [Talos Linux](https://www.talos.dev/) for running applications. This is a short overview of how it is put together.

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

## Networking

**OPNsense** runs as the firewall and router for the network.

## Self-hosted applications

Everything below runs as workloads on the Talos Kubernetes cluster:

- **Nextcloud** – files, calendar and contacts
- **Immich** – photo library
- **Vaultwarden** – password manager
- **Home Assistant** – home automation
- **Navidrome** – music streaming
- **Homebox** – inventory
- **n8n** and **Node-RED** – automation and workflows
- **NocoDB** – database front-end

## Why this setup

Talos removes most of the maintenance that comes with running a general-purpose Linux distribution under Kubernetes: there is no SSH, no package manager and no config drift, just a declarative machine configuration. Combined with Proxmox for virtualisation and TrueNAS for storage, it gives me a homelab that is straightforward to reason about and quick to recover when something breaks.
