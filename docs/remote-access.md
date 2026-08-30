# Remote access via Cloudflare Tunnel

A few services need to be reachable from outside the home network — mainly Jellyfin (so media can be watched away from home) and Jellyseerr (so requests can be made remotely). Rather than forwarding ports on the router, remote access goes through a **Cloudflare Tunnel**.

## How it works

The `cloudflared` container runs inside LXC 101 alongside the rest of the Docker stack and holds an outbound-only connection to Cloudflare's network, authenticated with a tunnel token. Cloudflare then routes public hostnames to the tunnel, which forwards requests to the right internal service — in this setup:

- `jellyfin.yourdomain.com` → Jellyfin 
- `requests.yourdomain.com` → Jellyseerr 

(`yourdomain.com` is a placeholder — the real config uses an actual domain managed in Cloudflare.)

## Why a tunnel instead of port forwarding

- **No open inbound ports.** The router never needs a port forward or DMZ rule — the connection is always initiated outbound from `cloudflared` to Cloudflare, so there's no listening port on the public IP for anything to scan or attack directly.
- **No dependency on a static/dynamic public IP.** Since nothing points directly at the home IP, a dynamic-DNS setup isn't needed either.
- **Cloudflare sits in front.** Traffic passes through Cloudflare's network before reaching the tunnel, which brings in their DDoS protection and TLS termination for free.

## Secrets

The tunnel is authenticated with a `TUNNEL_TOKEN` — this is effectively a credential that lets Cloudflare route traffic to this specific tunnel, so it's treated the same as any other secret and redacted from this repo. See [`lessons-learned.md`](lessons-learned.md).
