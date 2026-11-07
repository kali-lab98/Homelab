# Exposing a public service from an isolated LXC

Everything else in this lab is LAN-only or reached through a Cloudflare Tunnel (see [`remote-access.md`](remote-access.md)). One service is different: it's a volunteer node in a community-run distributed image cache, and the network's servers have to connect to it directly on a raw TCP port with its own TLS certificate. A tunnel can't do that, so this is the lab's first router port forward. The LAN is flat (no VLANs), so this doc covers how the container is fenced off from everything else.

*Status: running since November 2026. The service passed the upstream network's external connectivity test on its first start.*

## Threat model in one line

If the service (a Java process taking connections from the whole internet) is ever compromised, the attacker should end up in a small container that can reach the internet and nothing on the home network: not the Proxmox UI, not the file share, not DNS.

## Layout

| Piece | Choice | Why |
|---|---|---|
| Guest | Dedicated unprivileged LXC (Debian 13, 2 vCPU, 1 GB RAM) | Own blast radius. Root in the CT is an unprivileged UID on the host |
| Cache disk | 450 GB volume on the NVMe (`local-lvm`), excluded from backups | SSD suits lots of small random reads. The USB ZFS pool can mount late at boot, which would show the app an empty cache |
| Network | One NIC on the LAN bridge with the Proxmox firewall flag on; public DNS resolvers | Pi-hole lives on the LAN, which this container isn't allowed to reach |
| Exposure | Router forwards one TCP port to the container | The only inbound path |

## The firewall

Proxmox's firewall has three levels: datacenter, node and guest. Guest rules only take effect once the datacenter switch is on. Turning that switch on with default-DROP policies is the classic way to lock yourself out of the web UI. So:

1. **Datacenter: enabled, with ACCEPT in and out.** The host and every other container behave exactly as before. This step only "arms" the guest rules.
2. **Guest (this container only): default DROP inbound**, with:

```
IN  ACCEPT -p tcp -dport <service-port>          # from anywhere
IN  ACCEPT -source <lan-cidr> -p icmp            # ping for troubleshooting
OUT DROP   -dest <lan-cidr>                       # no new connections into the LAN
# policy_out ACCEPT: the internet stays reachable; ipfilter on: it can't spoof another IP
```

Replies to connections the LAN starts still pass (conntrack ESTABLISHED/RELATED runs first). The container just can't *start* a conversation with anything on the LAN.

3. **Dead-man switch.** Before flipping the datacenter switch:

```bash
systemd-run --on-active=10min --unit=fw-deadman pvesh set /cluster/firewall/options --enable 0
pvesh set /cluster/firewall/options --enable 1 --policy_in ACCEPT --policy_out ACCEPT
# ...verify from a NEW session...
systemctl stop fw-deadman.timer
```

If anything locks me out, the firewall turns itself off 10 minutes later. It's the same idea as `commit confirmed` on network gear.

**Verified before exposing anything:**

| From | Test | Result |
|---|---|---|
| Inside the container | internet (`curl ifconfig.me`), DNS, upstream API | works |
| Inside the container | Proxmox UI, Pi-hole, Jellyfin, the router | `curl` 000 / ping 100% loss |
| Desktop on the LAN | every existing service (Proxmox, SMB, Cockpit, Portainer, Pi-hole DNS, Jellyfin, Grafana) | still up |
| Other containers | outbound internet; Cloudflare tunnel connector | unaffected, 0 errors |

Then, before starting the real service, a throwaway 60-second listener on the service port logged who connected. A connection to the **public** IP showed up from the router's LAN address (NAT loopback). That proves the forward lands on the right container and that the router isn't keeping the port for one of its own features.

## The service: a sandboxed systemd unit

The app is a single Java jar, so it runs as a plain systemd service rather than in Docker:

```ini
User=<app>                        # dedicated system user, no shell
AmbientCapabilities=CAP_NET_BIND_SERVICE
CapabilityBoundingSet=CAP_NET_BIND_SERVICE   # can bind 443, can do nothing else root-like
NoNewPrivileges=true
ProtectSystem=strict              # whole OS read-only...
ReadWritePaths=<data> <logs> <tmp> <cache>   # ...except its own dirs
ProtectHome=true
PrivateTmp=true
StandardInput=null                # never waits on a console prompt
SuccessExitStatus=143             # SIGTERM → the app logs out cleanly → JVM exits 143
Restart=on-failure
RestartSec=300
StartLimitBurst=5                 # don't hammer upstream on a fatal error
```

`ProtectSystem` and friends need mount namespaces, which an unprivileged LXC only allows with `nesting=1`. Without it, the unit fails with `226/NAMESPACE`.

## Details that mattered

- **The credential file must not end in a newline.** The app reads `ID-KEY` from a file and doesn't trim it. A file written with an editor gets a trailing `\n`, the key fails validation, and the app falls back to a console prompt that never comes. It's written with `read -s` plus `printf '%s-%s'`, which also keeps it out of shell history.
- **The incoming and outgoing IP must match.** The upstream servers check that the node they connect to is the one that called them, so this container must not route through the VPN container the rest of the stack uses.
- **CGNAT check first:** the router's WAN IP has to match `curl ifconfig.me`, or no port forward will ever work.
- **The cache volume is excluded from backups.** A restore brings back an empty cache, and that's the one case where resetting the cache upstream is legitimate.

## What broke along the way


- **`tune2fs` refused a mounted volume** with `MMP: device currently active`. Proxmox enables ext4 multi-mount protection on container volumes. Stopping the container for a few seconds was enough to drop the 5% root reserve on the cache disk, which got back 22 GB
- **Passphrase-protected key and non-interactive SSH.** `ssh -v` said `Server accepts key`, but login still failed, because `BatchMode` can't ask for the passphrase. The Windows `ssh-agent` service ships *disabled*. Enabling it and running `ssh-add` once fixed it.
- **A failed login exits cleanly.** With `StandardInput=null`, a bad credential file makes the app hit EOF at its login prompt and exit with status 0. systemd sees success and doesn't restart it. Good to know when "inactive" shows up with no error.

## What I'd change / next steps

- Finish audit item M5: host rules (8006/22 from the LAN only), then host policy DROP, then 2FA.
- Move this container to its own VLAN/DMZ when the network gets segmented. The guest firewall is the stand-in until then.
- A blackbox TCP probe from Prometheus ([`monitoring.md`](monitoring.md)) so an outage alerts instead of quietly eating the uptime budget.

