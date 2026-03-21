# Port Reference — All Nodes

All listening TCP/UDP ports confirmed from system audits on 2026-03-20.

---

## josiahlab (192.168.0.28)

| Port | Proto | Bind | Process | Service |
|---|---|---|---|---|
| 22 | TCP | 0.0.0.0 / :: | sshd | OpenSSH |
| 53 | TCP/UDP | 0.0.0.0 / :: | docker-proxy | AdGuard Home — DNS |
| 80 | TCP | * | casaos-gateway | CasaOS web UI |
| 139 | TCP | 0.0.0.0 | smbd | Samba NetBIOS |
| 445 | TCP | 0.0.0.0 / :: | smbd | Samba SMB |
| 784 | UDP | 0.0.0.0 / :: | docker-proxy | AdGuard — DNS-over-QUIC |
| 853 | TCP | 0.0.0.0 / :: | docker-proxy | AdGuard — DNS-over-TLS |
| 3001 | TCP | 0.0.0.0 / :: | docker-proxy | AdGuard web UI |
| 5201 | TCP | * | iperf3 | iperf3 server |
| 5252 | TCP | * | tailscale | Tailscale web UI |
| 5900 | TCP | 127.0.0.1 | qemu-system-x86 | KVM VNC console (local only) |
| 9090 | TCP | * | josiah-api | Custom application — undocumented |
| 11434 | TCP | * | ollama | Ollama LLM API |
| 18789 | TCP | 0.0.0.0 / :: | docker-proxy | OpenClaw Gateway |
| 18790 | TCP | 0.0.0.0 / :: | docker-proxy | OpenClaw Gateway |
| 37932 | UDP | 0.0.0.0 | tailscaled | Tailscale WireGuard |

---

## CT100 — services (192.168.0.26)

| Port | Proto | Bind | Process | Service |
|---|---|---|---|---|
| 22 | TCP | 0.0.0.0 / :: | sshd | OpenSSH |
| 25 | TCP | 127.0.0.1 / ::1 | master (Postfix) | Postfix MTA — loopback only |
| 3000 | TCP | 0.0.0.0 / :: | docker-proxy | Grafana |
| 9000 | TCP | 0.0.0.0 / :: | docker-proxy | Portainer HTTP |
| 9090 | TCP | 0.0.0.0 / :: | docker-proxy | Prometheus |
| 9100 | TCP | * | prometheus-node | Node Exporter |
| 9443 | TCP | 0.0.0.0 / :: | docker-proxy | Portainer HTTPS |
| 41641 | UDP | 0.0.0.0 / :: | tailscaled | Tailscale WireGuard |

---

## DESKTOP-T6EKV7E (192.168.0.37) — Windows

| Port | Proto | Bind | Service |
|---|---|---|---|
| 22 | TCP | 0.0.0.0 / :: | OpenSSH |
| 135 | TCP | 0.0.0.0 / :: | RPC Endpoint Mapper |
| 139 | TCP | 192.168.0.37 | NetBIOS Session |
| 445 | TCP | :: | SMB |
| 3260 | TCP | 192.168.0.37, 100.95.230.4, 127.0.0.1 | iSCSI — StarWind |
| 3261 | TCP | 0.0.0.0 | iSCSI secondary |
| 3333 | TCP | 127.0.0.1 | StarWind Management Console |
| 5040 | TCP | 0.0.0.0 | Windows internal |
| 5357 | TCP | :: | WS-Discovery |
| 5432 | TCP | 0.0.0.0 / :: | PostgreSQL |
| 5988 | TCP | 0.0.0.0 / :: | WBEM / CIM — StarWind SMI-S |
| 8811 | TCP | :: | Unidentified |
| 9000 | TCP | :: | Unidentified |
| 9991 | TCP | :: | Unidentified |
| 11434 | TCP | :: | Ollama |
| 12372 | TCP | :: | Unidentified |
| 27036 | TCP | 0.0.0.0 | Steam Remote Play |
| 27060 | TCP | 127.0.0.1 | Steam — local |
| 47984 | TCP | 0.0.0.0 | Sunshine |
| 47989 | TCP | 0.0.0.0 | Sunshine |
| 47990 | TCP | 0.0.0.0 | Sunshine |
| 48010 | TCP | 0.0.0.0 | Sunshine |
| 41196 | TCP | 100.95.230.4 | Tailscale internal |
| 43303 | TCP | fd7a:... | Tailscale internal |
