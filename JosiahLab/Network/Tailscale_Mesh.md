# Tailscale Mesh VPN

**Tailscale domain:** taild067bc.ts.net
**IPv4 range:** 100.64.0.0/10
**IPv6 range:** fd7a:115c:a1e0::/48
**MagicDNS server:** 100.100.100.100

---

## Why Tailscale

Tailscale replaced what would otherwise be a complicated mess of port forwarding rules, dynamic DNS configuration, and VPN certificate management. The D5001 is an ISP-provided modem with limited firewall configurability, and punching holes through it for every service that needs remote access is both tedious and a security liability. Tailscale's approach — WireGuard tunnels between all registered devices, coordinated through a central control plane — means remote access is just a matter of being enrolled in the same Tailscale account. No port forwarding, no DDNS, no certificate renewal.

The tradeoff is the dependency on Tailscale's control plane. If Tailscale's coordination servers are unreachable, new connections cannot be established (though existing direct WireGuard connections between peers may persist for a while depending on the situation). For a homelab this is an acceptable dependency. The alternative — running a self-hosted WireGuard setup — would provide more control but requires significantly more maintenance and carries its own failure modes.

---

## Registered Lab Nodes

| Node | IPv4 | IPv6 | Deployment |
|---|---|---|---|
| DESKTOP-T6EKV7E | 100.95.230.4 | — | Windows client |
| josiahlab | 100.106.123.9 | fd7a:115c:a1e0::bc01:7b0a | Docker container |
| services (CT100) | 100.64.59.1 | fd7a:115c:a1e0::13a:3b01 | Native systemd |

Five additional nodes appear in the routing tables of the lab machines:

| IP | Identity |
|---|---|
| 100.83.123.2 | Unknown — phone, laptop, or remote machine |
| 100.90.93.19 | Unknown |
| 100.101.125.82 | Unknown |
| 100.108.106.52 | Unknown |
| 100.115.120.49 | Unknown |

These are all enrolled in the same Tailscale account and are being routed to but not identified in this documentation. The Tailscale admin console at `https://login.tailscale.com/admin/machines` would show names and last-seen times for all of them.

---

## Deployment — josiahlab (Container)

Tailscale on josiahlab runs inside a Docker container managed by CasaOS. This is a slightly unorthodox approach — Tailscale typically runs as a native daemon — but it works because the container uses host networking mode, meaning it shares josiahlab's network stack directly rather than being isolated behind a Docker bridge.

The container needs two things that are not normally available inside Docker: the `/dev/net/tun` device (for creating the WireGuard TUN interface) and the ability to modify the host's routing table. Both are provided via bind mounts and the host network mode.

```yaml
container_name: tailscale
image: tailscale/tailscale:v1.90.8
network_mode: host
entrypoint:
  - /bin/sh
  - -c
  - tailscaled --state=/var/lib/tailscale/tailscaled.state & sleep 10; tailscale web --listen 0.0.0.0:5252
environment:
  TS_STATE_DIR: /var/lib/tailscale
volumes:
  - /DATA/AppData/<appid>:/var/lib/tailscale
  - /dev/net/tun:/dev/net/tun
restart: unless-stopped
```

The entrypoint starts `tailscaled` in the background, waits 10 seconds for it to initialise, then starts `tailscale web` — the browser-based management interface on port 5252. This 10-second sleep is a simple but fragile way to handle startup ordering. If the daemon takes longer than 10 seconds to be ready (for example on a slow boot), the web UI may fail to start. A more robust approach would be to use a proper readiness check or have the web UI retry rather than relying on a sleep timer.

State is persisted in CasaOS's AppData directory, which means Tailscale keeps its authentication across container restarts. CasaOS app updates that recreate the container will preserve auth as long as the AppData directory is not wiped.

---

## Deployment — CT100 (Native systemd)

CT100 runs Tailscale as a native systemd service (`tailscaled.service`). This is the more conventional approach and is more reliable than the Docker container variant. Tailscale on CT100 also takes ownership of DNS configuration — it rewrites `/etc/resolv.conf` to use MagicDNS (100.100.100.100) and adds the `taild067bc.ts.net` search domain.

```
nameserver 100.100.100.100
search taild067bc.ts.net
```

This means any DNS lookup from CT100 goes through Tailscale's resolver first. Short hostnames like `josiahlab` resolve to their Tailscale IPs because of the search domain. CT100's own hostname is `services.taild067bc.ts.net` — resolvable from any other Tailscale node.

The consequence is that CT100's DNS does not go through the AdGuard Home instance on josiahlab. This is a minor inconsistency in the lab — most machines use AdGuard for DNS filtering, but CT100 bypasses it. If DNS-level ad/tracker blocking is wanted for CT100 as well, one option is configuring Tailscale to use a custom DNS resolver (pointing back at AdGuard's Tailscale IP rather than MagicDNS).

---

## Firewall Integration

Tailscale injects its own iptables rules on both Linux nodes to handle traffic marking and routing. These rules are managed entirely by the Tailscale daemon and should not be edited manually.

The relevant behaviour is:
- Traffic arriving on `tailscale0` is accepted and forwarded
- Traffic claiming to be from the CGNAT range (100.64.0.0/10) but arriving on a non-Tailscale interface is dropped
- Tailscale-originated traffic is marked and allowed to forward through the routing table

CT100 uses the standard WireGuard port `41641/udp`. josiahlab uses `37932/udp` — Tailscale assigns ports per-node based on what is available and stable for that machine.

---

## Services Accessible via Tailscale

Because there is no firewall between the Tailscale IPs and the services on each node (UFW is inactive), every listening port on every node is reachable from any other Tailscale node. The full list is in [[Network/Port_Reference]]. The more notable ones:

| Service | Node | Tailscale URL |
|---|---|---|
| AdGuard Home | josiahlab | http://100.106.123.9:3001 |
| CasaOS | josiahlab | http://100.106.123.9 |
| Ollama | josiahlab | http://100.106.123.9:11434 |
| Grafana | CT100 | http://100.64.59.1:3000 |
| Portainer | CT100 | http://100.64.59.1:9000 |
| Prometheus | CT100 | http://100.64.59.1:9090 |
| Ollama (GPU) | DESKTOP | http://100.95.230.4:11434 |
| iSCSI | DESKTOP | 100.95.230.4:3260 |

---

## Version Registry

| Node | Version | Method |
|---|---|---|
| josiahlab | v1.90.8 | Docker container |
| CT100 | Check with `tailscale version` | systemd daemon |
| DESKTOP-T6EKV7E | Check Tailscale system tray | Windows client |

---

## Future Plans

The current Tailscale setup works well for the basic remote access use case. Some extensions worth considering:

**Subnet routing:** josiahlab could be configured as a Tailscale subnet router, advertising `192.168.0.0/24` to the mesh. This would allow Tailscale nodes to reach any LAN device (not just ones with Tailscale installed) through josiahlab as a relay. This is useful for accessing devices like the GS108E management UI or the D5001 management page from outside the LAN.

**Exit node:** The Windows machine or josiahlab could be configured as a Tailscale exit node, routing all internet traffic from other Tailscale clients through the home connection. Useful when on an untrusted network.

**Access controls (ACLs):** The current setup allows every Tailscale node to reach every other node on every port. Tailscale ACLs can restrict which nodes can talk to which, and on which ports. As the mesh grows, this becomes more important — a compromised phone should not have unrestricted access to the iSCSI targets or the Prometheus instance.
