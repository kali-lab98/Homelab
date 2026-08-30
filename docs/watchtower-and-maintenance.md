# Keeping things updated: Watchtower & deunhealth

Two small helper containers keep the Docker stack low-maintenance without needing to manually `docker compose pull && up -d` every service.

## Watchtower

[Watchtower](https://containrrr.dev/watchtower/) watches every other running container and pulls a fresh image when a new one is published, then recreates the container with it.

Configuration used here:

- **Poll interval:** `86400` seconds (once every 24 hours)
- **Cleanup:** enabled — old images are removed after a successful update, so disk doesn't slowly fill with stale image layers
- **Include stopped / revive stopped:** both disabled — Watchtower only touches containers that are currently running, and won't restart ones that were deliberately stopped
- **Notifications:** sends an email (via Gmail SMTP) on every update, at `debug` log level, so update activity is visible without having to check container logs

**Secrets note:** the real configuration includes an SMTP username/password (a Gmail app password). That's redacted from this repo — see [`lessons-learned.md`](lessons-learned.md).

## deunhealth

[deunhealth](https://github.com/qdm12/deunhealth) runs alongside Watchtower and does a narrower job: it watches Docker's built-in healthcheck status for containers labeled `deunhealth.restart.on.unhealthy=true` (qBittorrent, in this stack) and restarts them if they go unhealthy — for example, if gluetun's VPN tunnel drops and qBittorrent's healthcheck (a ping test) starts failing.

It runs with `network_mode: none` (it doesn't need network access itself) and only needs the Docker socket mounted in, to be able to see container health status and issue restarts.

## Why both

Watchtower handles "the software is out of date," deunhealth handles "the software is currently broken." Together they cover most day-to-day maintenance without manual intervention — the main thing still worth checking manually is whether an image update introduced a breaking config change, which neither tool can detect on its own.
