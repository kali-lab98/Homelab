# The Servarr stack

Everything below runs as Docker containers inside LXC 101 (`servarr`), managed through Portainer (`:9443`). This is the classic "*arr" media automation stack: indexers find content, the *arr apps decide what to grab, qBittorrent downloads it, and Jellyseerr lets other people on the network request things without touching any of the backend UIs.

## Services

| Service | Port | Role |
|---|---|---|
| gluetun | 8080, 6881, 9696 (proxied) | VPN gateway container — routes qBittorrent and Prowlarr's traffic through a WireGuard tunnel |
| deunhealth | — | Watches Docker healthchecks and restarts any container that goes unhealthy |
| qBittorrent | 8080 | Torrent client |
| Prowlarr | 9696 | Indexer manager — one place to configure trackers, shared with Sonarr/Radarr/Lidarr |
| Sonarr | 8989 | TV show management (matches releases to indexers, sends to qBittorrent) |
| Radarr | 7878 | Movie management (same idea, for movies) |
| Lidarr | 8686 | Music management |
| Bazarr | 6767 | Fetches subtitles for Sonarr/Radarr's library |
| FlareSolverr | 8191 | Solves Cloudflare/CAPTCHA challenges on behalf of Prowlarr for indexers that need it |
| Jellyseerr | 5055 | Request front-end — lets other users request movies/shows without needing access to Sonarr/Radarr directly |

## Why only some services go through the VPN

Only **qBittorrent** and **Prowlarr** are configured with `network_mode: service:gluetun`, meaning they share gluetun's network namespace and all their traffic (including qBittorrent's actual torrent traffic) is forced through the WireGuard tunnel. If gluetun's healthcheck fails, those two containers have no working network at all, by design — this is what stops torrent traffic from ever going out over the raw ISP connection.

Everything else (Sonarr, Radarr, Lidarr, Bazarr, Jellyseerr, FlareSolverr) runs on the regular Docker bridge network, unprotected by the VPN. That's intentional, not an oversight: these apps only talk to indexers' search APIs, to each other, and to clients on the LAN — none of them handle actual torrent traffic, so there's nothing sensitive to tunnel. Routing everything through the VPN would add unnecessary latency and a single point of failure for services that don't need it.

FlareSolverr specifically is pinned to `network_mode: bridge` (not gluetun) so that indexers relying on it stay reachable from Prowlarr regardless of the VPN's state.

## Startup ordering

qBittorrent and Prowlarr both declare `depends_on: gluetun` with `condition: service_healthy` — Docker Compose won't start them until gluetun's healthcheck (a simple ping test) passes. Without this, both containers would start with gluetun's network namespace before the VPN tunnel is actually up, and traffic would either fail or (worse) briefly leak outside the tunnel.

## Shared storage

All the media/download-handling containers mount the same `/data` volume (the 24TB ZFS-backed mount from the host) so that when qBittorrent finishes a download, Sonarr/Radarr/Lidarr can find and import it without any file copying between containers.

## Secrets

The real Compose file for this stack contains a WireGuard private key (for gluetun) and would contain VPN provider credentials. None of that is included here — see [`lessons-learned.md`](lessons-learned.md) for how secrets are handled in this repo.
