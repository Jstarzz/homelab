# DESKTOP-T6EKV7E — Windows Services

**Host:** DESKTOP-T6EKV7E — 192.168.0.37 — Windows 10 21H2
**Tailscale IP:** 100.95.230.4

---

## StarWind Virtual SAN

### Background and Purpose

StarWind VSAN is the piece that connects the Windows machine's storage to the rest of the lab. It presents the 1 TB and 2 TB HDDs as iSCSI targets — block devices that other machines on the network can mount as if they were locally attached drives. josiahlab mounts these targets via the Linux iSCSI initiator (`open-iscsi`), and the resulting block devices can be used directly for KVM VM disk images, for Docker volumes, or for anything else that benefits from having more storage than the NVMe drive alone provides.

The motivation for this setup is that josiahlab has a 238 GB NVMe drive that fills up quickly once KVM VM disks and Docker overlay filesystems start accumulating. The Windows machine has 3 TB of HDD storage that was largely sitting idle as local Windows drives. StarWind turns that idle storage into shared infrastructure. It is a reasonable way to pool resources across machines that already exist rather than buying dedicated NAS hardware.

The Windows machine was chosen as the storage provider rather than the Proxmox node because it is more powerful, has more disk capacity, and the StarWind software stack integrates naturally with Windows. StarWind VSAN is a Windows-native product — it runs as Windows services, is configured through a Windows management console, and fits naturally into the Windows ecosystem.

### How it is configured

Three of the four StarWind services are running. The NVMe-oF service is stopped and unconfigured.

| Service | Status | Role |
|---|---|---|
| StarWindService | Running | Core engine — hosts the iSCSI targets |
| StarWindClusterService | Running | HA cluster coordination (active even with a single node) |
| StarWindSMISAgent | Running | WBEM/CIM interface for external storage management tools |
| StarWindNVMeoF | Stopped | NVMe over Fabrics — not configured |

The cluster service being active does not necessarily mean there is a multi-node StarWind cluster. It handles cluster-related functions even in a single-node setup.

### iSCSI binding

StarWind is listening for iSCSI connections on three addresses:

| Address | Port | Notes |
|---|---|---|
| 192.168.0.37 | 3260 | LAN iSCSI — used by josiahlab |
| 100.95.230.4 | 3260 | Tailscale iSCSI — remote access possible |
| 127.0.0.1 | 3260 | Loopback |
| 0.0.0.0 | 3261 | Secondary iSCSI port |
| 127.0.0.1 | 3333 | StarWind Management Console — local only |

The Tailscale binding (100.95.230.4:3260) is interesting. It means an iSCSI initiator on any Tailscale node could theoretically mount a StarWind target over the WireGuard tunnel. In practice, iSCSI over WireGuard would have higher latency and lower throughput than LAN iSCSI, so it is mainly useful as a maintenance or recovery option rather than a production connection path. Whether this binding is intentional or just a consequence of StarWind binding to all available interfaces is worth confirming.

### StarWind configuration snapshot

| Parameter | Value | Notes |
|---|---|---|
| Log level | 1 — errors only | Raise to 3 for debugging |
| Log rotate size | 100 MB | — |
| Log files kept | 20 | — |
| Update check interval | 7 days | Calls home to starwindsoftware.com |
| Copy ID | D055479A-6CAD-4437-A9CD-0AE6BDE44833 | License tracking |
| Last update check | 2026-03-14 | — |
| VAAI Extended Copy | Enabled | VMware hardware offload — no effect with KVM |
| VAAI Compare-and-Write | Enabled | Same — VMware specific |
| VAAI Write Same | Enabled | Same |
| ODX | Disabled | Windows Server Offloaded Data Transfer |
| WU Service Control | Enabled | StarWind manages Windows Update timing |

The VAAI features are VMware-specific storage offload capabilities. Since josiahlab uses KVM rather than VMware, these have no practical effect. They are enabled by default and are harmless to leave enabled.

WU Service Control being enabled means StarWind is preventing Windows Update from restarting the machine at unexpected times while iSCSI targets may be in use. This is sensible behaviour — an unexpected reboot would drop all active iSCSI connections, which could corrupt a KVM VM disk that was in the middle of writing.

### NVMe-oF

The NVMe-oF service exists but is stopped and shows `CanStop: False`, indicating it has not been configured. NVMe over Fabrics is a newer storage protocol that offers lower latency than iSCSI by removing some of the TCP/IP overhead. On a local GbE network the performance difference is unlikely to be noticeable, and iSCSI is well-understood and well-supported by the Linux iSCSI initiator stack. There is no immediate need to enable NVMe-oF.

---

## Ollama on GPU

Running Ollama on the Windows machine is specifically about the RTX 3080. josiahlab's i3-1115G4 has integrated Tiger Lake graphics, and while it can run Ollama inference, performance is limited compared to GPU inference. The RTX 3080 has 10 GB GDDR6X VRAM and CUDA support — it can run 7B models entirely in VRAM with room to spare, and inference speed is dramatically faster than CPU-only execution.

The practical use of this is twofold. First, any application or script that needs fast LLM inference points its API calls at `http://192.168.0.37:11434` (or `http://100.95.230.4:11434` via Tailscale) rather than the josiahlab endpoint. Second, heavier models that do not fit on the integrated graphics can run here where there is real VRAM.

The current problem is that Ollama is bound to all interfaces with no authentication. Anyone on the LAN or Tailscale mesh can submit inference requests and consume the GPU. This has not caused problems yet because the mesh is trusted devices, but it is not a principled approach.

```
Port: :11434 (all interfaces via ::)
GPU: RTX 3080, 10 GB GDDR6X
Authentication: none
```

---

## Sunshine — Game and Desktop Streaming

Sunshine is an open-source GPU-accelerated desktop streaming server. It works with Moonlight clients on any platform — Android, iOS, Windows, Linux, macOS, Apple TV. The RTX 3080's NVENC encoder handles video compression, so streaming is hardware-accelerated and does not meaningfully load the CPU.

The four ports in the 47984–48010 range are Sunshine's standard listening ports for different functions: RTSP for session negotiation, video/audio streams, and control input.

```
Ports: 47984, 47989, 47990, 48010 (TCP, 0.0.0.0)
GPU: RTX 3080 NVENC
Config UI: https://localhost:47990
```

With Tailscale active, Sunshine is accessible from outside the LAN without any port forwarding. The streaming quality over a WireGuard tunnel depends heavily on the available bandwidth and latency, but within the local network over GbE it is effectively full-quality, low-latency, and indistinguishable from sitting at the machine.

One practical use is remote desktop access to the Windows machine when away from home — Moonlight provides a much lower-latency experience than RDP for interactive use, especially for anything graphical.

---

## PostgreSQL

PostgreSQL is running and listening on all interfaces. Its purpose has not been identified in this audit. Possible candidates: a local application that needed a database (some monitoring tools, automation scripts, or productivity apps use PostgreSQL), or StarWind's management database (some versions use external databases), or something else entirely that was installed and forgotten.

```
Port: :5432 (0.0.0.0 and ::)
Status: Running
Purpose: Unknown — needs investigation
```

The binding to all interfaces is the pressing issue. Unless something on another machine genuinely needs to connect to this PostgreSQL instance, it should be restricted to localhost. The practical risk is that any device on the LAN can attempt authentication against it — the security depends entirely on the PostgreSQL authentication configuration (pg_hba.conf) being correctly set.

---

## OpenSSH Server

Windows' built-in OpenSSH server is active on port 22. This allows SSH sessions into the Windows machine from any LAN device or Tailscale node using standard SSH clients. PowerShell is the default shell for SSH sessions on Windows.

```
Port: :22 (0.0.0.0 and ::)
Status: Running
Default shell: PowerShell
```

SSH is useful for scripted management, file transfers via `scp` or `sftp`, and running commands on the Windows machine from Linux hosts without needing RDP. Combined with Tailscale, it provides SSH access from anywhere on the mesh without port forwarding.

---

## Steam and Remote Play

Steam is running with Remote Play active (port 27036). This enables streaming Windows games to other devices on the LAN via Steam Link or the Steam app. It is somewhat redundant with Sunshine (which does the same thing more flexibly and with hardware encoding), but both are active simultaneously without conflict.

---

## Unidentified Ports

Four ports are listening on the Windows machine but have not been attributed to a known service:

| Port | Protocol | Bind |
|---|---|---|
| 8811 | TCP | :: |
| 9000 | TCP | :: |
| 9991 | TCP | :: |
| 12372 | TCP | :: |

These need to be identified. Running services on a machine that also serves iSCSI storage to other lab nodes means an unidentified listener represents an unknown attack surface. Port 9000 is conventionally Portainer, SonarQube, or various other services — but Portainer is on CT100, not Windows.

---

## Future Plans

The most impactful hardware change for this machine is the RAM upgrade — moving from single-channel 16 GB to at minimum dual-channel, and ideally to a higher total capacity given the platform's support for much more.

On the software side, setting up Ollama properly as a controlled GPU inference endpoint — restricting it to localhost and exposing it selectively via a proxy with auth — would make the RTX 3080 a much more intentional resource in the lab's compute architecture rather than an open API anyone on the network can hit.

The Sunshine setup is already good for the streaming use case. Expanding it to explicitly support remote work scenarios (remote desktop from a laptop via Tailscale) rather than just gaming would extend its value considerably.

Identifying the four unknown port listeners and the PostgreSQL database owner are both tasks that should happen before the next security review.

---

## Proof of Work

- [ ] StarWind Management Console — targets, connected initiators, and virtual disk sizes
- [ ] Sunshine config UI — apps configured for streaming
- [ ] Task Manager GPU tab — Ollama utilisation during inference
- [ ] PostgreSQL — identify the owning application and review pg_hba.conf
- [ ] Identify the four unknown port listeners (8811, 9000, 9991, 12372)
- [ ] Windows Firewall — all three profiles correctly enabled
- [ ] Windows Event Viewer — StarWind application log entries
