# Proxmox host & storage

## The hardware

The whole homelab runs on a single repurposed laptop rather than a purpose-built server:

- **ASUS ROG Zephyrus 14"** (WQXGA 120Hz panel, not that it matters for a headless server)
- **CPU:** Ryzen 9 5900HS (8 cores / 16 threads, with integrated Radeon graphics)
- **RAM:** 16GB
- **GPU:** NVIDIA GeForce RTX 3060 Laptop GPU (passed through to the Jellyfin container for transcoding)

Storage is external: a **CENMATE 4-bay USB enclosure** (2.5"/3.5" SATA, USB-A/C 3.0, hot-swappable). RAID is handled entirely in software via ZFS, not by the enclosure.

## Hypervisor: Proxmox VE 9.2

Proxmox runs on bare metal and hosts everything else as LXC containers (not full VMs — lighter weight, appropriate for a laptop with 16GB of RAM shared across several services).

- Proxmox web UI: `:8006`
- Originally installed as 8.4.19, upgraded to 9.2 in September 2026 once 8.x support lapsed — see [`proxmox-upgrade-8-to-9.md`](proxmox-upgrade-8-to-9.md) for the full process, what the pre-flight checker caught, and what to expect right after rebooting.

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

Six containers run on top of Proxmox:

| ID | Name | Purpose |
|---|---|---|
| 100 | Cockpit | Web-based Linux admin UI, used here to serve a network share to the Windows desktop |
| 101 | servarr | Docker host (via Portainer) running the media automation stack |
| 102 | pihole | Network-wide DNS and ad-blocking |
| 103 | jellyfin | Media server, with GPU passthrough for hardware-accelerated transcoding |
| 104 | monitoring | Docker host for Prometheus + Grafana. See [`monitoring.md`](monitoring.md) |
| 105 | public cache node | The only internet-facing service. Firewalled off from the LAN. See [`isolated-public-service-lxc.md`](isolated-public-service-lxc.md) |

The two newer containers (104, 105) keep their disks on the NVMe (`local-lvm`) rather than the USB pool: metrics and cache writes stay off the USB disks, they get real snapshots in backups, and they can't start before the pool is mounted at boot.

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
