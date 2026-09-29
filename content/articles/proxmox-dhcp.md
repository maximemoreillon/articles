---
date: "2026-09-29T00:00:00+09:00"
title: "Running a Proxmox host on DHCP"
tags: ["Homelab", "Proxmox"]
---

Proxmox expects a static IP address, and its installer sets one up. On a network without control over the router, nothing guarantees that this address stays valid: if the router's subnet changes, the host is left without a gateway, without internet access and, in my case, without Tailscale, so without remote access either. DHCP avoids that, but Proxmox needs an adjustment to work with it.

## The problem

Switching the bridge to DHCP is simple: keep its port, drop `address` and `gateway`:

```
auto vmbr0
iface vmbr0 inet dhcp
        bridge-ports eno1
        bridge-stp off
        bridge-fd 0
```

But the host's name is also mapped to its address in `/etc/hosts`:

```
192.168.1.10 pve.example.lan pve
```

`pve-cluster`, the service behind `/etc/pve` (VM definitions, storage, users, the web UI's certificates), resolves the host's name at startup, and refuses to start if it doesn't resolve to an address configured on the host. Once the DHCP address differs from the one in `/etc/hosts`, the next reboot leaves Proxmox without its configuration: no web UI, no `qm` commands and no VM autostart.

There are two ways to keep `/etc/hosts` valid. Both are meant for a standalone host: in a Proxmox cluster, nodes reach each other through their configured addresses, which shouldn't change.

## Option 1: update `/etc/hosts` from DHCP

A `dhclient` exit hook can rewrite the line whenever a lease is obtained:

```bash
# /etc/dhcp/dhclient-exit-hooks.d/update-etc-hosts
case "$reason" in
  BOUND|RENEW|REBIND|REBOOT)
    sed -i "s/^.*\spve.example.lan\s.*$/${new_ip_address} pve.example.lan pve/" /etc/hosts
    ;;
esac
```

- `/etc/hosts` always holds the real LAN address, so the web UI and anything that hands the node's address to clients (SPICE, for instance) use a reachable address.
- If DHCP fails at boot, for instance because the router is down or the subnet just changed, `/etc/hosts` still holds the previous lease, which the host no longer has, and `pve-cluster` doesn't start.
- It depends on `dhclient`, which is end-of-life, and on the `sed` pattern matching the line. Either one failing goes unnoticed until a reboot.

## Option 2: point the name at a static address that can't change

The name only needs to resolve to *some* non-loopback address of the host, not necessarily the LAN one. A bridge without any physical port can carry a static private address:

```
auto vmbr1
iface vmbr1 inet static
        address 10.10.30.111/24
        bridge-ports none
        bridge-stp off
        bridge-fd 0
```

```
10.10.30.111 pve.example.lan pve
```

- `pve-cluster` starts whatever happens on the LAN, even without any DHCP server, and there is no script to maintain.
- Proxmox considers that address its own: the web UI shows it, and features that hand the node's address to clients may give out an address they can't reach. The noVNC console isn't affected, since it goes through the web UI's connection.
- It needs a bridge, although an otherwise unused one is enough.

I chose the second option. On my network, the whole point of DHCP is to survive changes to the network's settings, which is precisely when the first option can fail. In my case, the bridge already existed as a [private storage network between my VMs](/articles/proxmox-storage-bridge/). On a network where DHCP is reliable, the first option is the better fit.

## Applying it remotely

A mistake in `/etc/network/interfaces` can cut off a remote host, so I armed an automatic rollback before applying anything:

```bash
cp -a /etc/network/interfaces /root/interfaces.bak
cp -a /etc/hosts /root/hosts.bak
# edit both files, then:
systemd-run --on-active=300 --unit=net-revert /bin/sh -c \
  'cp -a /root/interfaces.bak /etc/network/interfaces; cp -a /root/hosts.bak /etc/hosts; ifreload -a'
ifreload -a
```

If the connection doesn't come back, the old files are restored after five minutes. `systemd` runs the rollback, so it doesn't depend on the SSH session that may have been cut.

`pve-cluster` only reads `/etc/hosts` at startup, so the running service says nothing about the next boot. Restarting it tests the new file while the rollback is still armed, without affecting the running VMs:

```bash
systemctl restart pve-cluster pveproxy
systemctl is-active pve-cluster pveproxy
```

If both are active and the host has its DHCP address, the rollback can be cancelled:

```bash
systemctl stop net-revert.timer
```
