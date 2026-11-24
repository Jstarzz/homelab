# JosiahLab — Homelab Documentation

HOME

Last audited: 2026-03-20 | Vault version: 3.0.0

> [!warning] Historical snapshot
> This vault describes the home lab as of March 2026. The November 2026 inventory of the Proxmox servers is in the [repository README](../README.md).

---

## What This Is

This is a living record of a personal homelab that has grown organically over time — not according to some master plan, but in response to whatever problem needed solving at a given moment. The result is a mix of hardware that was repurposed, upgraded, or acquired specifically for the lab, running a stack of services that have accumulated layer by layer. This document exists partly as reference, partly as a forcing function to actually understand what is running and why, and partly so that if everything burned down tomorrow, it could be rebuilt without starting from scratch mentally.

The lab is not trying to be enterprise infrastructure. It is trying to be reliable enough to depend on daily, flexible enough to experiment with, and documented well enough that future decisions can be made with context rather than guesswork.

---

## Physical Network Path

There is no dedicated hardware router in this lab. The ISP line goes directly into a D5001 modem/gateway combo, which handles NAT, DHCP, and routing for the entire 192.168.0.0/24 subnet. From there, a Netgear GS108E v2 eight-port managed switch distributes connectivity to all the lab nodes. The switch has QoS and IGMP snooping enabled, port mirroring active (useful for packet capture), but no VLANs configured. Everything is flat.

```
ISP
  └── D5001 Modem/Gateway (192.168.0.1 / 192.168.0.254)
        └── GS108E v2 (8-port managed switch)
              ├── DESKTOP-T6EKV7E   192.168.0.37
              ├── josiahlab         192.168.0.28
              ├── PVE Node          192.168.0.?
              └── Other LAN devices
```

The lack of a dedicated router/firewall appliance is the most significant architectural gap in the current setup. The D5001 is an ISP-issued consumer device with limited configurability. All perimeter defence falls on the host-based firewalls on each machine — and as of the March 2026 audit, UFW is inactive on both Linux nodes and two Windows Firewall profiles are disabled. This is acknowledged and sits at the top of the things-to-fix list.

The longer-term plan is to introduce a small x86 machine running pfSense or OPNsense between the D5001 and the switch, which would give proper firewall ruleset control, VLAN capability, IDS/IPS, and a place to terminate VPN connections at the network level rather than per-host.

---

## Node Overview

| Hostname | Role | LAN IP | Tailscale IP | OS |
|---|---|---|---|---|
| D5001 | Modem / gateway | 192.168.0.1 | — | ISP proprietary |
| DESKTOP-T6EKV7E | Primary workstation / iSCSI target | 192.168.0.37 | 100.95.230.4 | Windows 10 21H2 |
| josiahlab | Hypervisor / app server / AI node | 192.168.0.28 | 100.106.123.9 | Ubuntu 24.04.4 LTS |
| services (CT100) | Monitoring / Docker host | 192.168.0.26 | 100.64.59.1 | Ubuntu 24.04.4 LTS (PVE LXC) |

---

## Remote Access Strategy

Remote access runs entirely over Tailscale. There is no port forwarding through the D5001, and no services are intentionally internet-exposed. Tailscale provides WireGuard-based mesh connectivity between all lab nodes and any other registered device — phones, laptops, or remote machines. This makes the setup reasonably portable: connecting to the lab from anywhere is just a matter of being on the Tailscale network.

The tradeoff with this approach is that Tailscale is a dependency. If Tailscale is down or a node's auth key expires, remote access goes away. There is no fallback tunnel. This is acceptable for a homelab context but worth keeping in mind.

---

## Current Service Health (2026-03-20)

| Service | Host | Port | Status |
|---|---|---|---|
| AdGuard Home | josiahlab | :3001 UI / :53 DNS | Up 11 days |
| Ollama | josiahlab | :11434 | Running |
| CasaOS | josiahlab | :80 | Running |
| OpenClaw Gateway | josiahlab | :18789-18790 | Up 29 hrs |
| OpenClaw CLI | josiahlab | — | Crashed — exit code 1 |
| Tailscale container | josiahlab | host-net | Up 10 days |
| Portainer CE | CT100 | :9000 / :9443 | Up 3 days |
| Grafana | CT100 | :3000 | Up 2 days |
| Prometheus | CT100 | :9090 | Up 2 days |
| Node Exporter | CT100 | :9100 | Running |
| StarWind VSAN | DESKTOP-T6EKV7E | :3260 iSCSI | Running |
| StarWind NVMe-oF | DESKTOP-T6EKV7E | — | Stopped |

---

## Vault Index

- [[Hardware/DESKTOP-T6EKV7E]]
- [[Hardware/josiahlab]]
- [[Hardware/PVE-Node]]
- [[Hardware/GS108E-Switch]]
- [[Network/Network_Topology]]
- [[Network/Tailscale_Mesh]]
- [[Network/Port_Reference]]
- [[Services/josiahlab_Services]]
- [[Services/CT100_Services]]
- [[Services/Windows_Services]]
- [[Security/Security_Audit]]
- [[Troubleshooting/Troubleshooting_Ledger]]

---

## Rebuild Checklist

If starting from scratch:

- [ ] Restore Tailscale auth keys for all nodes
- [ ] Re-apply netplan static IP on josiahlab (192.168.0.28/24)
- [ ] Re-deploy AdGuard Home container on josiahlab
- [ ] Re-configure StarWind VSAN targets on Windows workstation
- [ ] Re-deploy monitoring stack (/opt/monitoring/docker-compose.yml) on CT100
- [ ] Re-deploy Portainer on CT100
- [ ] Restore CasaOS and app data from /DATA/ backup
- [ ] Re-enable Ollama service on josiahlab
- [ ] Restore rclone remote configuration and re-enable the service
- [ ] Re-import KVM VM definitions on josiahlab
- [ ] Re-enable Ollama on Windows and verify GPU is being used
