# CT100 (services) — Services

**Host:** services — 192.168.0.26 — Ubuntu 24.04.4 LTS (Proxmox LXC)
**Container ID:** CT100
**Tailscale FQDN:** services.taild067bc.ts.net
**Docker:** 29.3.0 | Compose: v5.1.0
**Uptime at audit:** 3 days, 1h 30m
**Kernel:** 6.17.2-1-pve (shared Proxmox host kernel)

---

## What This Container Is For

CT100 exists as a dedicated home for infrastructure monitoring. The decision to put the monitoring stack in a separate Proxmox container rather than on josiahlab was deliberate — a monitoring system that lives on the same machine as most of what it monitors is somewhat self-defeating. If josiahlab has a problem and goes down or becomes unresponsive, the monitoring dashboards should still be reachable. Keeping Grafana and Prometheus on a separate container on a separate physical machine means they are independent of josiahlab's state.

CT100 is a lightweight LXC container with 4 GB RAM allocated and a 63 GB root disk. At audit time only 3.7 GB of the disk was in use (7%) and 383 MB of RAM was used with no swap active — a very comfortable operating state compared to the memory pressure on josiahlab.

The choice of LXC over a full VM for this workload makes sense. LXC containers share the host kernel, start faster, have less overhead, and are appropriate for services that do not need kernel-level isolation. Docker runs inside CT100, which requires some specific Proxmox configuration to enable — LXC containers need `nesting=1` and `keyctl=1` in their feature flags to support Docker's namespacing requirements.

---

## Running Containers

```
CONTAINER ID   NAME                      IMAGE                            STATUS
34d57416d634   monitoring-prometheus-1   prom/prometheus                  Up 2 days
bc2f06a0e675   monitoring-grafana-1      grafana/grafana                  Up 2 days
09e417f30de3   portainer                 portainer/portainer-ce:latest    Up 3 days
```

All three containers are healthy and have been running for multiple days without issues at the time of the audit.

---

## Portainer CE

Portainer is the Docker management UI. It provides a browser-based interface for viewing containers, images, volumes, networks, and compose stacks — essentially a visual layer on top of Docker commands. On CT100 it manages the monitoring stack and any other containers that might be deployed here in the future.

```
Container: portainer
Image: portainer/portainer-ce:latest
Status: Up 3 days
Network: docker0 bridge (172.17.0.2)
Ports: :9000 (HTTP), :9443 (HTTPS)
Data: portainer_data (named Docker volume)
Socket: /var/run/docker.sock (read-only bind mount)
```

Portainer accesses Docker via the Unix socket in read-only mode for most operations, but needs write access to manage containers. The `portainer_data` volume holds Portainer's own database — user accounts, environment configurations, stack definitions. This volume should be included in any backup strategy for CT100.

The admin password is stored as a bcrypt hash in the original compose definition. If the plaintext was set during initial deployment and not recorded, it can be reset via the Portainer CLI (`portainer --reset-password`) after stopping the container.

Access: `http://192.168.0.26:9000` or `https://192.168.0.26:9443`

---

## Grafana

Grafana is the dashboarding and visualisation layer. It connects to Prometheus as a data source and renders time-series metrics as graphs, tables, heatmaps, and other visualisations. The dashboards can show anything that Prometheus is scraping — CPU usage, memory, disk I/O, network traffic, container health, custom application metrics.

```
Container: monitoring-grafana-1
Image: grafana/grafana (latest at deploy time)
Status: Up 2 days
Network: monitoring_default (172.18.0.2)
Port: :3000
Data: monitoring_grafana_data (named Docker volume)
```

The `monitoring_grafana_data` volume stores Grafana's SQLite database, which contains the dashboard definitions, alert rules, data source configurations, and user accounts. Losing this volume means losing all custom dashboards and having to recreate them from scratch — it is worth backing up.

Grafana and Prometheus are on the same Docker network (`monitoring_default`, 172.18.0.0/16) rather than the default bridge. This is the right approach — they can communicate with each other directly by container name without going through the host network stack, and they are isolated from other containers that might be on the default bridge.

Access: `http://192.168.0.26:3000`

Default credentials on first run: `admin` / `admin`. Grafana forces a password change on first login.

---

## Prometheus

Prometheus is the metrics collection and storage backend. It scrapes metrics from configured targets at regular intervals and stores them in a time-series database (its own embedded TSDB). Grafana then queries Prometheus to build dashboards.

```
Container: monitoring-prometheus-1
Image: prom/prometheus (latest at deploy time)
Status: Up 2 days
Network: monitoring_default (172.18.0.3)
Port: :9090
Data: monitoring_prometheus_data (named Docker volume)
Config: ./prometheus.yml (bind-mounted from host)
```

The Prometheus configuration file lives at `/opt/monitoring/prometheus.yml` on the CT100 host and is bind-mounted into the container. This means the config can be edited on the host and then reloaded without rebuilding the container. The `monitoring_prometheus_data` volume holds the TSDB data — historical metrics are stored here and are retained according to Prometheus's default retention policy (15 days unless changed in the config).

### Compose file

The monitoring stack is defined at `/opt/monitoring/docker-compose.yml`:

```yaml
services:
  prometheus:
    image: prom/prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    ports:
      - "9090:9090"
    restart: unless-stopped

  grafana:
    image: grafana/grafana
    volumes:
      - grafana_data:/var/lib/grafana
    ports:
      - "3000:3000"
    restart: unless-stopped

volumes:
  prometheus_data:
  grafana_data:
```

The compose file is minimal and does not define a custom network — both containers end up on the automatically-created `monitoring_default` bridge network, which is the correct default behaviour for a compose project. They can reach each other by service name (`prometheus`, `grafana`).

The Prometheus scrape configuration (`prometheus.yml`) was not captured in the audit. This is a significant gap — without knowing what targets Prometheus is scraping, it is not possible to know what metrics are being collected or whether the monitoring is actually complete. The file needs to be captured and documented here.

Access: `http://192.168.0.26:9090`

---

## Prometheus Node Exporter

Node Exporter is the one service on CT100 that runs natively rather than in a container. It is a Prometheus exporter that exposes hardware and OS-level metrics — CPU utilisation, memory, disk I/O, network interface statistics, filesystem usage — in the format that Prometheus can scrape.

```
Service: prometheus-node-exporter.service
Port: :9100 (all interfaces, TCP)
Status: Running
```

Running Node Exporter natively rather than in a container is the right choice for this type of exporter. Container-based Node Exporter deployments need various privilege escalations and namespace configurations to actually see the host's metrics rather than the container's view of metrics. The native daemon sees everything cleanly.

Node Exporter is presumably in Prometheus's scrape configuration targeting `localhost:9100` or `192.168.0.26:9100`. The metrics it provides would cover CT100's own resource usage. Whether any other machines (josiahlab, the Windows workstation) are also being scraped by this Prometheus instance is not captured — that depends on the `prometheus.yml` content.

---

## Tailscale (native)

Tailscale runs as a native systemd daemon on CT100, which is the more conventional approach compared to the Docker container on josiahlab. The daemon manages the WireGuard tunnel and handles DNS configuration by writing to `/etc/resolv.conf`.

```
Service: tailscaled.service
Port: 41641/udp
Status: Running
IP: 100.64.59.1
```

CT100's DNS points to Tailscale's MagicDNS resolver (100.100.100.100) rather than AdGuard Home. This is a consequence of Tailscale taking ownership of DNS when it starts. It is functional but means CT100's DNS traffic bypasses AdGuard's filtering and logging.

---

## Postfix

Postfix is running but only listening on loopback interfaces (`127.0.0.1` and `::1`). This is not a mail server — it is a local relay for system-generated emails (cron job output, alert notifications, etc.). Nothing external can send to it or receive from it. It is likely delivering to a remote relay (an ISP smarthost or a service like SendGrid) for any emails the system generates.

The relay configuration was not captured. If CT100 is generating any system alerts or automated emails, understanding where they go is worth documenting.

---

## Docker Volumes

| Volume | Used By | Notes |
|---|---|---|
| monitoring_grafana_data | Grafana | Dashboards, SQLite DB, alert rules |
| monitoring_prometheus_data | Prometheus | TSDB — historical metrics data |
| portainer_data | Portainer | User accounts, environment configs, stack defs |
| 3b91c57a8579... (unnamed hash) | Unknown | Needs to be identified |

The unnamed volume with a hash-based name is unusual. Docker auto-generates names like this when a container defines a volume without a name in its compose file or run command. Identifying which container it belongs to requires inspecting volume metadata.

---

## Storage

| Filesystem | Type | Size | Used | Mount |
|---|---|---|---|---|
| /dev/mapper/vg_shared-vm--100--disk--0 | ext4 | 63 GB | 3.7 GB | / |

CT100 has generous disk headroom. The entire running state of three containers plus Prometheus TSDB history, Grafana dashboards, and Portainer configuration is using under 4 GB. The monitoring stack will grow as Prometheus accumulates metric history, but 63 GB is a comfortable ceiling for the foreseeable future.

---

## Future Plans

The monitoring stack covers CT100's own metrics via Node Exporter. The next natural step is expanding Prometheus's scrape configuration to include the other lab machines. josiahlab would need Node Exporter installed as a systemd service (or as a container) and added to Prometheus's targets. The Windows machine would need a Windows-specific exporter (windows_exporter) for the same purpose.

With metrics from all machines flowing into Prometheus and Grafana dashboards built on top of them, there would be a unified view of the entire lab's health — CPU, memory, disk, and network across josiahlab, CT100, and the Windows workstation in one place.

Alerting is also not configured. Grafana has an alerting system that can send notifications via email, Slack, webhooks, or various other channels when a metric crosses a threshold. Setting up alerts for obvious failure conditions (disk usage above 85%, swap usage above 50%, a container going down) would reduce the amount of manual checking needed.

---

## Proof of Work

- [ ] Grafana dashboard showing Node Exporter metrics — CPU, memory, disk, network
- [ ] Prometheus targets page (http://192.168.0.26:9090/targets) — all scrape targets shown as UP
- [ ] Portainer showing all three containers in their current state
- [ ] Contents of /opt/monitoring/prometheus.yml — full scrape configuration
- [ ] docker volume inspect monitoring_grafana_data — confirm data is persisting
- [ ] Identify the unnamed Docker volume (3b91c57a...)
