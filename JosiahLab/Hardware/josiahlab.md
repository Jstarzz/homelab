# josiahlab — Dell Latitude 3520

**OS:** Ubuntu 24.04.4 LTS
**LAN IP:** 192.168.0.28 (static, via netplan bridge)
**Tailscale IP:** 100.106.123.9
**Serial:** 5S6DBS3
**Primary roles:** KVM hypervisor, Docker/CasaOS host, DNS server, AI inference node

---

## The Machine

josiahlab is a Dell Latitude 3520 laptop — a corporate-grade business laptop that has been repurposed as a server. It is not an obvious choice for a homelab server, but it has some practical advantages: low idle power consumption, built-in UPS (the laptop battery), a reasonable thermal envelope, and the 11th Gen Intel Core i3-1115G4 has Tiger Lake's improved neural processing and a good instruction set for a low-power chip. VT-x is supported, which is what matters for running KVM guests.

The form factor tradeoff is that it is a laptop, so the upgrade path is limited. There are only two SODIMM slots and currently only one is populated. With 8 GB in single-channel configuration, the machine is running under constant memory pressure — more on that below. The storage is a 238 GB NVMe SSD with about half of its capacity allocated, and the rest of the LVM physical volume sitting unallocated and ready to be used.

This machine runs persistently. The lid is presumably closed, the WiFi adapter is disabled (wlp0s20f3 shows DOWN with no carrier), and it is connected to the GS108E via the wired Ethernet port, which is enslaved to a Linux bridge. It has been up for 24 days continuously at the time of the audit, which is representative of normal operation.

---

## Specs

| Component | Detail |
|---|---|
| Manufacturer | Dell Inc. |
| Model | Latitude 3520 |
| CPU | Intel Core i3-1115G4 — 2C/4T, 3.00 GHz base / 4.10 GHz boost, Tiger Lake |
| Virtualization | VT-x supported |
| Instruction extensions | AVX-512, AES-NI, AVX2 |
| L1d | 96 KiB (2 instances) |
| L1i | 64 KiB (2 instances) |
| L2 | 2.5 MiB (2 instances) |
| L3 | 6 MiB |
| RAM | 8 GB DDR4-3200, SODIMM — slot 2 empty |
| Max RAM | 32 GB (2× SODIMM slots) |
| Storage | 238.5 GB NVMe (nvme0n1) |
| Firmware | v1.43.2, 2025-09-01 |

---

## Memory Situation

At the time of the audit, memory usage looked like this:

| | Total | Used | Free | Available |
|---|---|---|---|---|
| RAM | 7.5 GiB | 5.8 GiB | 345 MiB | 1.7 GiB |
| Swap | 4.0 GiB | 1.9 GiB | 2.1 GiB | — |

Nearly 2 GiB of swap in active use with under 400 MiB of free RAM. The machine is consistently at the edge of its memory budget. The main consumers are: the running KVM VM (which takes whatever its allocation is), the Docker containers managed by CasaOS (AdGuard Home, Tailscale container, OpenClaw), Ollama (which holds model weights in memory after the first load), and libvirtd itself plus the rest of the system.

The KVM guest and Ollama together likely account for the majority of the pressure. Ollama is particularly aggressive about memory — once a model is loaded, it stays resident until Ollama decides to evict it or is restarted. On a machine with 8 GB total, having a 3–4 GB model loaded alongside a running VM and several containers is going to produce swap.

The fix is adding a second 8 GB DDR4-3200 SODIMM to the empty slot. This doubles total RAM to 16 GB and simultaneously enables dual-channel operation, doubling memory bandwidth. Both effects matter: more headroom for the current workload, and better throughput for everything that touches memory intensively (KVM I/O, Docker overlay reads, Ollama inference). The Latitude 3520 supports up to 32 GB total.

---

## Storage Layout

| Device | Size | Type | Filesystem | Mount |
|---|---|---|---|---|
| nvme0n1 | 238.5 GB | NVMe SSD | — | — |
| nvme0n1p1 | 1 GB | EFI | vfat | /boot/efi |
| nvme0n1p2 | 2 GB | Boot | ext4 | /boot |
| nvme0n1p3 | 235.4 GB | LVM PV | LVM2 | — |
| ubuntu--vg-ubuntu--lv | 100 GB | LVM LV | ext4 | / |

The root filesystem is at 49% — about 46 GB used of the 100 GB LV. The LVM physical volume is 235.4 GB but only 100 GB has been provisioned as the root logical volume. This leaves roughly 135 GB of LVM space unallocated, which is immediately useful for carving out additional LVs — dedicated volumes for KVM disk images, a separate CasaOS data volume, or a large scratch space for Docker build operations that would otherwise fill up the root filesystem.

The current situation where everything shares the 100 GB root LV is workable but not ideal. Docker's overlay filesystems, KVM disk images (if stored locally), and CasaOS app data all live in that same pool. It would be cleaner to have `/var/lib/libvirt/images` and `/DATA` on their own LVs.

---

## Networking and the Bridge

The networking setup is more involved than a typical laptop-as-server would be. enp43s0 (the physical NIC) is enslaved to br0, a Linux bridge. This means enp43s0 no longer has an IP address directly — br0 holds the IP (192.168.0.28/24) and presents itself to the network as a bridge. Any KVM guest that attaches a tap interface to br0 will appear directly on the LAN with its own IP from the D5001's DHCP pool, rather than being NATted behind josiahlab's IP.

This is the preferred setup for KVM guests that need proper LAN presence. The currently running VM (192.168.0.32 — identified by the QEMU MAC prefix 52:54:00) connects via vnet0 which is attached to br0, and from the perspective of everything else on the network it looks like a regular LAN machine.

The bridge is configured with STP disabled and forward-delay set to 0. STP (Spanning Tree Protocol) would normally prevent loops but adds a 30-second delay when a port comes up before it starts forwarding traffic. For a simple setup with no redundant paths, STP is unnecessary and the delay is annoying, so it is turned off.

```yaml
# /etc/netplan/50-cloud-init.yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp43s0:
      dhcp4: no
  bridges:
    br0:
      interfaces: [enp43s0]
      dhcp4: no
      addresses: [192.168.0.28/24]
      routes:
        - to: default
          via: 192.168.0.1
      nameservers:
        addresses: [1.1.1.1, 8.8.8.8]
      parameters:
        stp: false
        forward-delay: 0
```

Docker adds its own bridge networks on top of this — docker0 for the default bridge network (172.17.0.0/16) and br-342fd1b4a627 for the openclaw_default network (172.18.0.0/16). Containers on these networks are NATted out through josiahlab's IP rather than appearing directly on the LAN, which is the standard Docker behaviour.

---

## CPU Vulnerability Notes

| Vulnerability | Status |
|---|---|
| Meltdown | Not affected — Tiger Lake architecture |
| Spectre v1 | Mitigated |
| Spectre v2 | Mitigated — Enhanced IBRS + PBRSB software sequence |
| MDS | Not affected |
| Retbleed | Not affected |
| Indirect Target Selection | Vulnerable — no fix available at this time |
| Spec Store Bypass | Mitigated via prctl |

The Indirect Target Selection vulnerability is the one outstanding unmitigated issue on this CPU. It is a relatively minor side-channel concern compared to the higher-profile vulnerabilities, and for a homelab context the practical risk is low — it would require an attacker to already be running code on the machine to exploit it.

---

## Future Plans

The RAM upgrade is the most impactful near-term change possible for this machine without replacing it. After that, the storage layout deserves a cleanup pass — carving dedicated LVs for VM storage and the CasaOS data directory would prevent the root filesystem from filling up unexpectedly.

Longer term, the question is whether a laptop makes sense as the primary hypervisor indefinitely. A small form factor x86 machine with more RAM slots, a proper PCIe slot, and better thermal headroom for sustained compute workloads would serve the hypervisor role better. The Latitude 3520 is currently pulling double duty as both the hypervisor and the application server because it is the only always-on Linux machine in the lab. Separating those roles — a small dedicated hypervisor node and a separate application server — would make the architecture cleaner and more resilient.

For now the Latitude 3520 is staying. It is doing the job, the power consumption is reasonable, and the remaining work is mostly software configuration rather than hardware limitations.

---

## Proof of Work

- [ ] htop showing CPU, RAM usage, and uptime under normal load
- [ ] df -h confirming storage layout and available LVM space (vgs / lvs)
- [ ] virsh list --all showing running VMs
- [ ] docker ps showing all containers
- [ ] ollama list showing loaded models
- [ ] free -h to confirm swap usage has improved after any RAM upgrade
