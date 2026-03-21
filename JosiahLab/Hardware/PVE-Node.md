# PVE Node — Proxmox VE Hypervisor

**Kernel (observed in CT100):** 6.17.2-1-pve (Proxmox build, 2025-10-21)
**Known containers:** CT100 (services) at 192.168.0.26
**Status:** Active at audit time

> [!note] Incomplete data
> The Proxmox host itself was not directly audited. This page is built from what can be inferred from CT100's audit output — the kernel version, visible block devices, CPU information, and MAC OUI of containers seen on the LAN. A direct audit of the PVE host is needed to fill in the gaps.

---

## Overview

There is a Proxmox VE node running somewhere on this network. Its existence is inferred from several signals: CT100 reports a kernel of `6.17.2-1-pve` (a Proxmox-patched kernel), its hostname is `services` with a Proxmox-format FQDN in `/etc/hosts` (`services.taild067bc.ts.net` with a `# --- BEGIN PVE ---` comment block), the block device `sda` of ~800 GB and an NVMe drive are visible inside the container, and the LVM volume group is named `vg_shared` providing the container's disk as `vg_shared-vm--100--disk--0` — the `vm-100-disk-0` naming is Proxmox's standard storage format for container ID 100.

The exact LAN IP of the Proxmox host is not confirmed from the audit data. Based on the ARP table entries with `bc:24:11` MAC prefixes (which is the Proxmox/Proxmox Virtual Environment OUI for bridged virtual NICs), there are at least three unknown devices on the LAN that are likely Proxmox containers or VMs: 192.168.0.39, 192.168.0.53, and 192.168.0.54.

---

## Inferred Hardware

| Component | Detail |
|---|---|
| CPU | Intel Core i3-7100U — 2C/4T, 2.40 GHz, Kaby Lake |
| Virtualization | VT-x supported |
| Physical storage | ~800 GB HDD (sda) + ~238.5 GB NVMe (nvme0n1) |
| Architecture | x86-64 |

The i3-7100U is a mobile dual-core Kaby Lake chip. It is not a powerful CPU for a hypervisor — 2 physical cores with hyperthreading gives 4 logical CPUs, and CT100 has 2 of those allocated (2 online, 2 offline at audit time). Running multiple containers and VMs on this hardware requires discipline about not over-allocating CPU resources.

### CPU Vulnerabilities

The i3-7100U has a significant vulnerability profile. Because it is an older consumer Kaby Lake chip, Intel has not released updated microcode that fixes several known side-channel vulnerabilities:

| Vulnerability | Status |
|---|---|
| MDS | Vulnerable — no microcode available |
| Retbleed | Vulnerable |
| Spec Store Bypass | Vulnerable |
| SRBDS | Vulnerable — no microcode |
| Old microcode flag | Vulnerable |
| Meltdown | Mitigated (PTI) |
| Spectre v1 | Mitigated |
| Spectre v2 | Mitigated (Retpolines) |

These vulnerabilities exist at the hardware level and are shared by every container and VM running on this host. There is no software fix available. This is a known constraint of running older consumer hardware as a hypervisor — server-grade hardware from the same era typically received more thorough microcode updates. The practical risk in a homelab context is low, but it is worth knowing.

---

## Storage Layout (inferred)

| Device | Size | Notes |
|---|---|---|
| nvme0n1 | ~238.5 GB | Likely Proxmox OS and LVM thin pool for VMs/CTs |
| nvme0n1p1 | 1007 KB | BIOS boot |
| nvme0n1p2 | 1 GB | Boot partition |
| nvme0n1p3 | ~237 GB | LVM PV |
| sda | ~800 GB | Bulk storage — likely additional Proxmox storage for larger disks |
| vg_shared LV for CT100 | 63 GB | CT100's root disk via LVM |

CT100 root at audit time: 3.7 GB used of 63 GB — plenty of headroom.

---

## Known Containers

| CT ID | Name | LAN IP | Tailscale IP | Role |
|---|---|---|---|---|
| 100 | services | 192.168.0.26 | 100.64.59.1 | Monitoring, Docker host |

Other Proxmox OUI (bc:24:11) MAC addresses seen on the LAN:

| IP | MAC | Identity |
|---|---|---|
| 192.168.0.53 | bc:24:11:b5:ba:00 | Unknown CT or VM |
| 192.168.0.54 | bc:24:11:6d:83:c7 | Unknown CT or VM |
| 192.168.0.39 | bc:f1:71:ec:f4:14 | Unknown |

---

## Proxmox Notes

Proxmox VE is a Debian-based hypervisor platform combining KVM for virtual machines and LXC for containers into a single management interface. CT100 is an LXC container, which means it shares the host kernel — hence the `6.17.2-1-pve` kernel visible inside the container. LXC containers are more lightweight than full VMs (no hardware emulation, no separate kernel) but they are more constrained in what they can do. The fact that Docker runs inside CT100 is slightly unusual — Docker inside LXC requires specific Proxmox configuration to allow the necessary kernel namespaces and capabilities.

The PVE web UI runs on port 8006 (HTTPS) on the host's IP. Access requires the host's IP to be known, which is currently undocumented.

---

## Future Plans

The i3-7100U is the bottleneck here, both in terms of core count and the microcode vulnerability situation. If the Proxmox node is going to host more containers or VMs, a more capable machine would serve the role better — ideally something with more cores, server-grade hardware for better microcode support, and ECC memory. A used Xeon E3 or E5 system would be a significant step up and is often affordable on the second-hand market.

On the software side, properly documenting all running containers and VMs on this node — not just CT100 — is overdue. The three unidentified Proxmox-OUI MAC addresses on the LAN suggest there are other workloads running that are not captured in this documentation.

---

## Proof of Work

- [ ] PVE web UI node summary — CPU, RAM, uptime, PVE version
- [ ] PVE storage view — pool names, sizes, and types
- [ ] Full CT/VM list with IDs, names, and assigned IPs
- [ ] pveversion output
- [ ] pvesm status output
- [ ] Host LAN IP documented
