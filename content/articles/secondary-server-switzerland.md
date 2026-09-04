---
date: "2025-08-15"
title: "Secondary server in Switzerland"
tags: ["Homelab", "Talos", "Kubernetes", "TrueNAS"]
---

I built a secondary server at my parents' house in Switzerland. It acts as an offsite target for TrueNAS replication and runs its own small Kubernetes cluster for workloads I want to keep running independently of the main homelab.

## Hardware

The setup mirrors my main homelab — [Proxmox](https://www.proxmox.com/) as the hypervisor, [Talos Linux](https://www.talos.dev/) for Kubernetes — but scaled down and built for a spare room rather than a rack:

- No rack-mount chassis, just a small tower case.
- A single pair of NVMe SSDs, mirrored, holding both the Proxmox boot install and the VM storage — unlike the main homelab, there's no separate SSD pool on the TrueNAS VM, only HDDs.

## Networking

Sensitive services such as the Proxmox UI and the Talos API are not exposed directly — they're only reachable over [Tailscale](https://tailscale.com/), which also links the server back to my main network so it can talk to the primary site.

## Storage and replication

As on the primary homelab, a TrueNAS VM manages the local disks. It's set up as a replication target: the primary TrueNAS pushes ZFS snapshots to it over the Tailscale link, so the offsite copy stays in sync without needing its own backup jobs.

## Workloads

Alongside the replication target, the server runs its own small Talos Kubernetes cluster for workloads that benefit from running off-site, including monitoring — so I still have visibility into the primary site if it goes down entirely, rather than losing monitoring along with everything else.

## Why this setup

Reusing the same Proxmox/Talos/TrueNAS stack as the main homelab means there's nothing new to learn to operate this box, while physically separating it at a different site closes the single-location gap that neither [democratic-csi's local replication nor Velero](/articles/velero/) could address on their own.
