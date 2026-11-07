# Lessons learned

## Handling secrets in a public repo

The original configuration files for this homelab contain real secrets:

- A WireGuard private key (gluetun/VPN)
- A Gmail SMTP app password (Watchtower's email notifications)
- A Cloudflare Tunnel token

None of these appear anywhere in this repo, and the domain name has been replaced with a placeholder (`yourdomain.com`) throughout. A few reasons that matters even for a "just documenting my homelab" repo:

- A leaked WireGuard key or tunnel token can be used by someone else to route traffic through your VPN account or Cloudflare tunnel.
- An SMTP app password gives access to send mail as that account.
- Even IPs and domains that look harmless on their own become more useful to an attacker once they're tied to a public list of exactly which services, versions, and container images are running behind them.
- GitHub history keeps everything forever by default — a secret committed once and later "removed" is still recoverable from an earlier commit unless the whole history is rewritten (and even then, assume it may have already been scraped by a bot — GitHub gets scanned for leaked credentials within minutes).

**Practical takeaway:** real secrets belong in a `.env` file (or Docker/Portainer secrets) that's listed in `.gitignore`, never in the compose file itself. Compose files can then reference `${VARIABLE_NAME}` and stay safe to commit.

## Architecture choices worth calling out

- **Repurposing an old laptop instead of buying server hardware** — a reasonable zero-cost way to get started, with the tradeoff that storage has to be external (USB DAS) rather than native SATA/SAS, which comes with its own reliability caveats (see [`proxmox-and-storage.md`](proxmox-and-storage.md)).
- **LXC over full VMs** — lighter on a laptop's limited RAM, at the cost of needing to hand-configure things like `nesting=1` for Docker-in-LXC and explicit device passthrough rules for GPU access, that a full VM would handle more transparently.
- **Scoping the VPN to only the containers that need it** (qBittorrent + Prowlarr) rather than tunneling the whole stack — keeps the rest of the *arr apps simple and avoids a VPN outage taking down services that don't touch torrent traffic.
- **Cloudflare Tunnel over port forwarding** — no inbound ports open on the router for remote access to the media apps. The one service that genuinely needs a raw port gets its own firewalled container instead of an exception for everything (see [`isolated-public-service-lxc.md`](isolated-public-service-lxc.md)).
- **Watchtower + deunhealth together** — a low-effort way to keep containers current and self-healing. Real metrics came later with Prometheus + Grafana ([`monitoring.md`](monitoring.md)), but self-healing and observability solve different problems, and both are worth having.

## Upgrading a live single-node hypervisor

A few things that held up during the Proxmox 8 → 9 major-version upgrade (full writeup: [`proxmox-upgrade-8-to-9.md`](proxmox-upgrade-8-to-9.md)):

- **A backup that's never been test-restored isn't a real safety net.** `apt dist-upgrade` across a major version has no downgrade path — the backups taken beforehand are the only way back, so they need to actually be trustworthy, not just present.
- **Proxmox's own pre-flight checker (`pve8to9`) catches things you wouldn't otherwise notice** — a stale `/etc/hosts` entry from before this box moved networks, and a leftover `systemd-boot` package that had nothing to do with how the system actually boots. Neither would have caused a problem day-to-day, but both were worth fixing before a major upgrade touched boot-related and networking packages.

## Opening the first inbound port

From putting one service directly on the internet (full write-up: [`isolated-public-service-lxc.md`](isolated-public-service-lxc.md)):

- **Arm a dead-man switch before touching a firewall.** `systemd-run --on-active=10min ... --enable 0` turns the firewall back off unless I cancel it after checking that I can still get in. It's cheap insurance against locking yourself out of a headless box.
- **Turn on the outer layer with ACCEPT policies first.** The Proxmox datacenter switch only arms guest rules, so enabling it with ACCEPT changed nothing for the host or the other guests while one container got default-deny.
- **Test both directions.** From inside the exposed container: the internet works and the LAN doesn't. From the LAN: every existing service still answers. Then a throwaway listener on the forwarded port proves the router sends traffic to the right place *before* the real service starts.
- **Re-check the router too.** Its own firewall turned out to be switched off. Opening a port is a good moment to audit the box doing the forwarding, not just the one receiving it.

## Monitoring

From building the Prometheus + Grafana stack ([`monitoring.md`](monitoring.md)):

- **Check the template architecture.** A `grep debian-13` on the template list matched an arm64 image first, and the container just failed to spawn. Filter on `amd64`.
- **Files copied from Windows can carry odd permissions.** `scp` from Windows created config folders as `drwx---rwx`. Linux checks group bits before "other", so Grafana (in group 0) was denied while Prometheus (only "other") worked fine. When one container can read a file and another can't, look at the group bits.

