# homelab

A small home lab I run for self-hosting, monitoring and local AI. Snapshot as of March 2026.

| Machine | Runs |
| --- | --- |
| Ubuntu 24.04 server | AdGuard Home (network DNS), Ollama, CasaOS, KVM guests |
| Proxmox VE node | Ubuntu LXC with Prometheus, Grafana, Node Exporter and Portainer |
| Windows workstation | StarWind VSAN iSCSI target, Ollama on the GPU |
| Netgear GS108E | Managed switch (QoS, IGMP snooping, port mirroring) |

- **Remote access:** Tailscale mesh between all nodes. No ports forwarded from the internet.
- **Monitoring:** Prometheus scrapes node exporters; Grafana dashboards; Portainer for containers.
- **Network:** a virtualized OpenWrt router does NAT and smart queue management (low latency under load, per-device priority), with automatic failover to a hardware router if it goes down.
- **Next:** VLAN segmentation and inter-VLAN firewall rules.

Detailed notes (inventory, network, audit findings) are kept private.
