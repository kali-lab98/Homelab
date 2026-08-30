# Proxmox host & storage

## The hardware

The whole homelab runs on a single repurposed laptop rather than a purpose-built server:

- **ASUS ROG Zephyrus 14"** (WQXGA 120Hz panel, not that it matters for a headless server)
- **CPU:** Ryzen 9 6900HS
- **RAM:** 16GB
- **GPU:** GTX 3070

Storage is external: a **CENMATE 4-bay USB enclosure** (2.5"/3.5" SATA, USB-A/C 3.0, hot-swappable). RAID is handled entirely in software via ZFS, not by the enclosure.

## Hypervisor: Proxmox VE 8.4.19

Proxmox runs on bare metal and hosts everything else as LXC containers (not full VMs — lighter weight, appropriate for a laptop with 16GB of RAM shared across several services).

- Proxmox web UI: `:8006`

## Storage: ZFS pool `vault`

The four drives in the USB enclosure are combined into a single ZFS pool:

```
pool: vault
state: ONLINE
config:
    vault       ONLINE
      raidz1-0  ONLINE
        sda     ONLINE
        sdb     ONLINE
        sdc     ONLINE
        sdd     ONLINE
```

`raidz1` tolerates one drive failure without data loss (similar to RAID 5). This pool backs the `Data` storage used by the LXC containers below (mounted into them as `/data`, 24TB).

**Worth knowing:** ZFS is normally run against drives with direct SATA/SAS access, where it can see each disk's real health and behavior. Running it over a USB DAS enclosure works, but USB adds a layer (bridge chipset, occasional dropouts/resets) that ZFS wasn't really designed to sit behind. It's a reasonable way to repurpose hardware you already own, but it's worth keeping backups of anything irreplaceable, and keeping an eye on `zpool status` for `read`/`write`/`cksum` errors that might trace back to USB flakiness rather than the drives themselves.

## LXC containers

Four containers run on top of Proxmox:

| ID | Name | Purpose |
|---|---|---|---|
| 100 | Cockpit | Web-based Linux admin UI, used here to serve a network share to the Windows desktop |
| 101 | servarr | Docker host (via Portainer) running the media automation stack |
| 102 | pihole | Network-wide DNS and ad-blocking |
| 103 | jellyfin | Media server, with GPU passthrough for hardware-accelerated transcoding |

### LXC 101 — servarr (Docker host)

Config highlights (from `/etc/pve/lxc/101.conf`):

- 4 cores, 4096MB RAM, 2048MB swap
- Unprivileged container, with `nesting=1` enabled (required for Docker-in-LXC)
- Two mount points: `/data` (24TB, backed by the `vault` ZFS pool — shared media/download storage) and `/docker` (128GB, local-lvm — container configs)
- A cgroup device rule and bind mount for `/dev/net/tun`, needed for the gluetun VPN container's WireGuard tunnel to work inside the LXC

See [`servarr-stack.md`](servarr-stack.md) for what actually runs inside this container.

### LXC 103 — jellyfin (GPU passthrough)

This container has extra device passthrough rules — bind mounts for `/dev/dri` (Intel/AMD render node) and `/dev/nvidia*` (NVIDIA), plus matching cgroup device allow rules. In an unprivileged LXC, hardware passthrough like this doesn't work by default — the container needs explicit `lxc.cgroup2.devices.allow` and `lxc.mount.entry` lines for every device node the GPU driver needs, added to the container config on the Proxmox host. This is what lets Jellyfin use NVENC hardware transcoding instead of falling back to (much slower) CPU transcoding.

### LXC 100 — Cockpit

Runs [Cockpit](https://cockpit-project.org/), a web-based server admin panel, used here mainly to expose a network share reachable from the Windows desktop on the LAN rather than for general Proxmox administration.
