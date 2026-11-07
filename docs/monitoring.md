# Monitoring: Prometheus + Grafana

Until October 2026 the only way to know how the host was doing was to open the Proxmox summary page and look at the last hour. This stack adds real metrics with history: Prometheus scrapes the hypervisor every 15 seconds and keeps 30 days, and Grafana turns that into dashboards (CPU, RAM, disks, ZFS, network, temperatures).

It's phase 1 of a longer plan. Alerting, per-container metrics and Proxmox API metrics come next (see the end).

## Layout

```
Proxmox host                         LXC 104 "monitoring" (Docker Compose)
  node_exporter :9100  <── scrape ──  Prometheus :9090  <── query ──  Grafana :3000  <── browser
                                         30d / 8 GB retention            provisioned data source
```

| Piece | Where it runs | Port | Notes |
|---|---|---|---|
| node_exporter | Proxmox host (Debian package `prometheus-node-exporter`) | `:9100` | Host metrics: CPU, memory, filesystems, ZFS, NICs, hwmon temps |
| Prometheus | LXC 104, Docker | `:9090` | Scrape interval 15s, retention 30 days **or** 8 GB, whichever comes first |
| Grafana | LXC 104, Docker | `:3000` | Admin password comes from `.env`; sign-up disabled; Prometheus data source provisioned from a file |

The container: unprivileged Debian 13, 2 vCPU / 2 GB RAM / 16 GB disk on the NVMe (`local-lvm`), with `nesting=1,keyctl=1` so Docker runs inside an unprivileged LXC. Compose file: [`compose-examples/monitoring-stack.yml`](../compose-examples/monitoring-stack.yml).

## Why it's built this way

- **Its own container, not the Docker host in LXC 101.** Monitoring should survive the thing it monitors. LXC 101 already runs a dozen containers and shares disks with two other LXCs, so if it has a bad day, the dashboards that would explain it shouldn't go down with it.
- **node_exporter on the host, not inside the LXCs.** Inside an LXC, node_exporter mostly reports the *host's* kernel numbers anyway. Running it once on the hypervisor gives the real picture: ZFS ARC, the USB disks, the physical NIC. Per-container numbers will come from the Proxmox API instead (phase 2).
- **NVMe, not the ZFS pool.** The time-series database writes constantly. Keeping it on `local-lvm` keeps that churn off the USB-attached pool and gives the container real snapshots for backups.
- **Retention capped by both time and size.** `--storage.tsdb.retention.time=30d` plus `--storage.tsdb.retention.size=8GB` means a burst of new metrics can't fill the 16 GB disk.
- **Grafana provisioning instead of clicking.** The data source is a YAML file mounted read-only into Grafana. Rebuilding the container gives the same setup with no manual steps:

```yaml
# grafana/provisioning/datasources/prometheus.yml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090   # Compose service name, resolved on the stack's network
    isDefault: true
```

- **Rejected:** a hosted Grafana/Prometheus service (cost, and a weaker learning story), and a full VM (RAM is the constraint on a 16 GB laptop).

## Scrape config

```yaml
# prometheus/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: prometheus          # Prometheus scraping itself: proves the pipeline works
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: node                # node_exporter on the Proxmox host
    static_configs:
      - targets: ["<proxmox-host>:9100"]
        labels:
          instance: homelab
```

The first dashboard is the community **Node Exporter Full** dashboard (Grafana ID `1860`). One query worth typing by hand, so the dashboard isn't magic:

```promql
100 * (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)   # host RAM used, %
```

That figure includes the ZFS ARC, which is why it reads higher than the Proxmox summary page.

## What broke

- **The container wouldn't start: `sync_wait ... Failed to spawn container`.** The template lookup (`pveam available | grep debian-13`) matched an **arm64** build first, and the log said `Detected container architecture: arm64` on an x86_64 host. Fix: filter on `amd64` explicitly, destroy, recreate.
- **Grafana showed "No data" on every panel, and the data source dropdown was empty.** `docker exec grafana ls /etc/grafana/provisioning/datasources` gave `Permission denied`. The config folders had been copied from Windows with `scp`, which created them as `drwx---rwx`: no access for the group, full access for "other". Grafana runs as uid 472 in **gid 0**, and Linux checks group permission bits *before* "other". The process matched the group, got nothing, and never fell through to the "other" bits. Prometheus runs as `nobody`, matched only "other", and could read its config fine, which made it confusing. Fix: `chmod -R u=rwX,go=rX` on the config folders, restart Grafana.
- **Lesson from that second one:** after copying configs from Windows, check modes with `ls -la` before starting anything.

## Status and next steps

Running since October 2026. Both scrape targets are up, and the stack wasn't affected when the Proxmox firewall was turned on in November (see [`isolated-public-service-lxc.md`](isolated-public-service-lxc.md)).

| Phase | What | State |
|---|---|---|
| 1 | Container, Prometheus, Grafana, host node_exporter, first dashboard | done |
| 2 | `prometheus-pve-exporter` with a read-only API token: per-guest CPU/RAM, storage, backup status | planned |
| 3 | cAdvisor in the Docker LXC: per-container metrics for the media stack | planned |
| 4 | Alerting (Grafana alerts or Alertmanager → email): disk filling, pool degraded, host down, backup missing | planned |
| 5 | Logs (Loki), a blackbox probe for the one internet-facing service, scraping a cloud VM over the tunnel | later |

Until phase 4 lands, this is *observability without alerting*: great for answering "what happened?", useless for "something is happening right now".
