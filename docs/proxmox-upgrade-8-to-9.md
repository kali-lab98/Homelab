# Proxmox VE 8 → 9 upgrade

Proxmox VE 8 (Debian 12 "Bookworm") support ended in mid-2026; Proxmox VE 9 (Debian 13 "Trixie") had been out and stable for over a year by then. This documents the actual upgrade of this homelab's single Proxmox node from 8.4.19 to 9.2.

## Before starting

Single-node upgrades are simpler than cluster ones (no HA/quorum to coordinate) but also have no fallback node — if something breaks mid-upgrade, the only way back is a restore, not a migration. Before touching anything:

- Confirmed fully patched on the current 8.x release (`apt update && apt dist-upgrade` reporting nothing to do)
- Ran Proxmox's own pre-flight checker, `pve8to9 --full`, and resolved every issue it raised
- Backed up `/etc/pve` (the cluster/VM/LXC config directory) and network config off the box
- Took a `vzdump` backup of every LXC container, stored somewhere other than the same storage pool the containers already live on
- Confirmed the ZFS pool backing container storage was healthy (`zpool status`)
- Picked a maintenance window, since the DNS container (Pi-hole) and the media server both had to go down for a while

## What the pre-flight checker caught

`pve8to9 --full` is worth taking seriously rather than skimming — it flagged two real failures and a couple of warnings on this system:

| Finding | Cause | Fix |
|---|---|---|
| Node hostname resolved to the wrong IP | A stale `/etc/hosts` entry left over from before this box moved to its current LAN | Corrected the IP in `/etc/hosts` |
| `systemd-boot` meta-package installed | Leftover package — the system actually boots via GRUB (confirmed with `efibootmgr -v`), not systemd-boot | Removed the unused package |
| `/etc/sysctl.conf` had a live setting | Debian 13 stops reading that file directly | Moved the setting into `/etc/sysctl.d/` |
| Missing CPU microcode package | Never installed in the first place | Installed the matching microcode package |

The hostname/IP one is a good example of why the checker matters even on a "nothing's changed" system — it's the kind of leftover that's invisible day-to-day but can cause real problems for internal Proxmox services after a major upgrade.

## The upgrade itself

With the pre-flight checks clean, the upgrade followed Proxmox's documented process: repoint the Debian and Proxmox APT repos from the old release to the new one, `apt update`, `apt dist-upgrade`, reboot into the new kernel.

A few things worth calling out for anyone doing this on similar hardware:

- **Shut down the containers before the package upgrade, not just before the reboot.** The LXC/container runtime packages themselves jump a major version during the upgrade — upgrading them underneath actively-running containers is asking for trouble.
- **Run the upgrade inside `tmux` (or `screen`), not a bare SSH session.** Networking packages get upgraded too, and if the network stack hiccups mid-upgrade, a plain SSH session dies — possibly leaving `apt` half-finished. A terminal multiplexer keeps the process alive on the host even if the SSH connection drops.
- **Don't run `apt autoremove` right after rebooting.** It would clear out the old kernel packages, which are exactly the fallback you want available in the boot menu until the new kernel is confirmed working.

## Verifying afterward

After reboot, checked in order: Proxmox's own version and running kernel, the ZFS pool's health, the network bridge, and then each service — DNS resolution through Pi-hole, the Docker stack's containers, the media server (including hardware transcode, not falling back to software), and the tunnel routing external traffic in.

## If something goes wrong

`apt dist-upgrade` isn't reversible — there's no downgrade path back to the previous major version. The real safety net is the backups taken beforehand: `/etc/pve` and the `vzdump` archives can restore individual containers, or the whole node if it comes to that. This is the reason those backups need to be more than just *taken* — a backup that's never been test-restored isn't a real safety net.
