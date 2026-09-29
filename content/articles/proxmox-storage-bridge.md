---
date: "2026-09-29T00:00:00+09:00"
title: "A host-only storage network to survive ISP network changes"
tags: ["Homelab", "Proxmox", "Talos", "TrueNAS", "Kubernetes"]
---

My [secondary server in Switzerland](/articles/secondary-server-switzerland/) runs Proxmox with two VMs: TrueNAS and a single-node Talos Kubernetes cluster, whose volumes are served by TrueNAS over iSCSI and NFS. The LAN is managed by the ISP's router, whose settings the ISP can override at any time.

One day, the ISP moved the LAN to a different subnet, losing the DHCP reservations along the way. The addresses the cluster used to reach TrueNAS no longer worked, so its volumes became unreachable.

The fix is to stop relying on the LAN for anything that must stay stable. TrueNAS and Talos run on the same Proxmox host, so they don't need the LAN to talk to each other: a Proxmox bridge without any physical port gives them a private network that no router can change.

## The bridge

On Proxmox, `vmbr1` has no physical port and no gateway. Jumbo frames cost nothing on a bridge that never leaves the host, so its MTU is 9000:

```
auto vmbr1
iface vmbr1 inet static
        address 10.10.30.111/24
        bridge-ports none
        bridge-stp off
        bridge-fd 0
        mtu 9000
```

Each VM gets a second VirtIO NIC on `vmbr1`, with the firewall unchecked (it only adds extra devices in the path) and the MTU left to "same as bridge". The addresses are static, since nothing else can hand them out: `.111` for Proxmox, `.112` for Talos and `.113` for TrueNAS. On TrueNAS, the new interface gets `10.10.30.113/24` and an MTU of 9000, without a gateway.

Jumbo frames only work if every member uses them, and a mismatch shows up as mounts that hang on large transfers. A ping at full size with fragmentation forbidden checks it:

```bash
ping -M do -s 8972 -c 3 10.10.30.113
```

## Talos

On Talos, the new NIC is configured in the machine config. The LAN interface keeps DHCP, so that it follows whatever the ISP decides:

```yaml
machine:
  network:
    interfaces:
      - deviceSelector:
          hardwareAddr: bc:24:11:xx:xx:xx # LAN
        dhcp: true
      - deviceSelector:
          hardwareAddr: bc:24:11:yy:yy:yy # storage bridge
        dhcp: false
        mtu: 9000
        addresses:
          - 10.10.30.112/24
```

There's a catch: Talos chooses the node's Kubernetes `InternalIP` and etcd's advertised address automatically among the node's addresses. Adding the NIC made it switch both to the new address. Rather than let a NIC change move them, I pinned them explicitly, and to the bridge, whose address exists as soon as the VM boots, regardless of the router or Tailscale:

```yaml
machine:
  kubelet:
    nodeIP:
      validSubnets:
        - 10.10.30.0/24
cluster:
  etcd:
    advertisedSubnets:
      - 10.10.30.0/24
```

I applied the change with `talosctl apply-config --mode=try --timeout 5m`, which rolls back automatically unless the config is applied again without `try`. That made it safe to change the network of a node I only reach remotely.

## Storage over the bridge

The CSI driver then only needs TrueNAS's bridge address. I used this as the opportunity to move from [democratic-csi](/articles/democratic-csi/) to [tns-csi](https://github.com/fenio/tns-csi), with `server: 10.10.30.113` in its storage classes.

For iSCSI, the address alone isn't enough. TrueNAS advertises every address an iSCSI portal listens on, and tns-csi logs into all of them. With the default portal listening on `0.0.0.0`, each volume ends up with extra sessions over the LAN and Tailscale. I created a second portal listening only on `10.10.30.113` and pointed tns-csi's storage class at it with `portalId`.

## Proxmox

The Proxmox host itself also moved to DHCP on its LAN bridge, so that it keeps internet access, and therefore Tailscale, whatever the subnet. That needs a small trick, which I describe in [a separate article](/articles/proxmox-dhcp/): the host's name must resolve to the static bridge address instead of the LAN one.

Proxmox backs up its VMs to an NFS share on TrueNAS, so that storage now uses `10.10.30.113` as well.

## Result

The LAN only provides internet access now, through DHCP, and the stable addresses live on networks I control: the host-only bridge for storage, and Tailscale for remote access. The next subnet change should only affect the router's port forwarding, which a tunnel will eventually replace.
