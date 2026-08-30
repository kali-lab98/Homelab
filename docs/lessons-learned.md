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
- **Cloudflare Tunnel over port forwarding** — no inbound ports open on the router at all for remote access.
- **Watchtower + deunhealth together** — a low-effort way to keep containers current and self-healing without a full monitoring stack.

## notes

