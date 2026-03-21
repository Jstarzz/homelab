# GS108E v2 — Netgear ProSAFE Plus Switch

**Model:** Netgear GS108E v2
**Ports:** 8x Gigabit Ethernet
**Role:** Core LAN switch connecting all nodes to the D5001 gateway

---

## Overview

The GS108E v2 is the backbone of the physical network. Every lab node connects through it. It sits between the D5001 modem and the rest of the machines, though it is doing nothing more sophisticated than forwarding frames — there are no VLANs, no link aggregation, no layer 3 functions. It is a managed switch being used like an unmanaged one, mostly. The management features that are enabled exist for specific reasons.

The switch was chosen because it was available and affordable. The GS108E line is one of the more accessible "managed" options that does not require a full enterprise switch to get features like port mirroring. For a homelab at this scale, eight ports is enough.

---

## Active Configuration

| Feature | State | Why |
|---|---|---|
| VLANs | Disabled | Not configured — the entire lab is flat on 192.168.0.0/24 |
| QoS | Enabled | Priority queuing, likely to favour iSCSI storage traffic |
| IGMP Snooping | Enabled | Reduces multicast flooding — relevant for mDNS and any multicast-based service discovery |
| Port Mirroring | Enabled | One port copies traffic from another for packet capture purposes |
| Jumbo Frames | Unknown | Worth checking — iSCSI performance benefits from frames larger than 1518 bytes |

### Port Mirroring

Port mirroring being enabled means traffic from one port is being copied to another — the intended use is connecting a device running Wireshark or tcpdump to the mirror destination port and being able to observe traffic from another port without physically intercepting the cable. This is a useful diagnostic tool but it also means a device on the mirror destination port passively sees all that traffic. Worth knowing which port is the destination and what is connected to it.

> [!todo] Document the mirror config
> Log into the GS108E management UI and record which port is the source and which is the destination, and whether it is ingress, egress, or both.

### QoS

QoS is active but the specific queue assignments have not been documented. The most likely use case is deprioritising bulk traffic relative to iSCSI or interactive traffic, given that the Windows workstation is running StarWind and serving iSCSI targets that other machines can mount. iSCSI is latency-sensitive and does not recover well from queue congestion on a shared link.

### IGMP Snooping

IGMP Snooping prevents multicast traffic from being flooded to every port on the switch. With Ollama running on two machines, mDNS active on several nodes, and Samba doing NetBIOS name resolution across the subnet, there is a reasonable amount of multicast and broadcast traffic on this network. Keeping it from being forwarded to every port is sensible.

---

## The Flat Network Problem

With no VLANs, all devices share one broadcast domain. The Windows workstation, the josiahlab hypervisor, the Proxmox node, its containers, and every other device on the network can reach each other without any routing or ACL in between. This is fine operationally but has security implications: a compromised device on the LAN has direct access to every service on every other device.

The future plan is to segment the network. Storage traffic (iSCSI between the workstation and josiahlab) should be on its own VLAN, ideally with dedicated ports and jumbo frames enabled. Lab/experimental traffic should be separated from anything more trusted. Services that do not need to talk to each other should not be able to. The GS108E v2 supports 802.1Q VLANs, so the hardware is capable — this is a configuration decision that has not been made yet.

---

## iSCSI Over a Shared Switch

iSCSI storage traffic between DESKTOP-T6EKV7E and josiahlab travels over the same physical switch as all other LAN traffic. In a production environment this would typically be on a dedicated network or at least a dedicated VLAN. In the current setup, a large file transfer or a backup job saturating a port could introduce latency on the iSCSI connection.

In practice this has not caused problems, but it is a known limitation. If storage performance becomes a concern, the most impactful changes would be: enabling jumbo frames on the switch and on both the iSCSI initiator and target interfaces, and eventually segregating storage onto a dedicated VLAN or a direct crossover link between the two machines.

---

## Physical Port Layout

> [!todo] Document port assignments
> Record which device connects to which switch port. This is essential for any future switch replacement, cable audit, or troubleshooting physical link issues.

| Port | Device | Speed Negotiated | Notes |
|---|---|---|---|
| 1 | | | |
| 2 | | | |
| 3 | | | |
| 4 | | | |
| 5 | | | |
| 6 | | | |
| 7 | | | |
| 8 | | | |

---

## Management Access

The GS108E v2 has a browser-based management interface. The switch can be found on the LAN either by its assigned IP (DHCP or static — not yet documented) or using the Netgear ProSAFE Plus utility for discovery.

> [!todo] Record the switch management IP
> Add it here so the UI can be reached without the discovery utility.

Default factory credentials: `admin` / `password`. If this has not been changed, it should be.

---

## Future Plans

The most significant planned change to the switch configuration is introducing VLANs once a proper edge firewall (pfSense or OPNsense) is in place. The intended topology would be:

- A management VLAN for access to switch, modem, and lab infrastructure
- A storage VLAN for iSCSI traffic between DESKTOP-T6EKV7E and josiahlab
- A general LAN VLAN for workstations and less sensitive devices
- Potentially a guest/IoT VLAN isolated from the rest

The GS108E supports all of this natively. It is purely a matter of designing the VLAN layout properly and configuring trunk ports for the machines that need to carry multiple VLANs (josiahlab in particular, since it runs VMs that may need to reach different segments).
