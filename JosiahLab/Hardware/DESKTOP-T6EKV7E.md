# DESKTOP-T6EKV7E — Primary Windows Workstation

**OS:** Windows 10 21H2, Build 19045, 64-bit
**LAN IP:** 192.168.0.37
**Tailscale IP:** 100.95.230.4
**Primary roles:** Workstation, iSCSI storage target, AI inference (GPU), game streaming

---

## The Machine

This is the main Windows workstation and arguably the most capable piece of hardware in the lab on raw specs. The Xeon E5-2697v3 is a 14-core/28-thread server-class processor on the Haswell-EP platform. It was originally a server or workstation chip, not a consumer desktop part — the LGA2011-v3 socket and the HEDT platform it lives on bring quad-channel memory support, a large core count, and the kind of I/O that server workloads expect. Paired with an RTX 3080, the machine is well-suited for anything that involves compute: Ollama inference, video work, game streaming via Sunshine, or just running a lot of applications simultaneously.

The weak point is the RAM. A single 16 GB DDR4-2133 DIMM means the machine is running in single-channel mode, and at 2133 MHz rather than the higher speeds the platform supports. For a 14-core processor, memory bandwidth is important — running single-channel cuts effective bandwidth roughly in half compared to dual or quad channel. The practical impact shows up in workloads that move a lot of data through memory, which includes both Ollama (loading model weights) and any situation where the CPU is doing parallel work across many cores. Adding even one more matched DIMM would be an immediate improvement. The platform supports up to 768 GB across all slots, so the ceiling is not a concern.

---

## Specs

| Component | Detail |
|---|---|
| CPU | Intel Xeon E5-2697 v3 — 14C/28T, 2.6 GHz base / 3.6 GHz boost, Haswell-EP, LGA2011-v3 |
| GPU | NVIDIA RTX 3080 Founders Edition — 10 GB GDDR6X |
| RAM | 16 GB DDR4-2133 — single DIMM, single-channel |
| Boot drive | ~120 GB SSD |
| HDD 1 | ~1 TB |
| HDD 2 | ~2 TB |
| Primary NIC | Realtek (MAC: 00:e0:4c:cd:06:d7) |

---

## Network

The machine has a static or DHCP-assigned LAN IP of 192.168.0.37 and is on the Tailscale mesh at 100.95.230.4. The Tailscale node is accessible from any other lab machine and from any personal device enrolled in the same Tailscale account, which makes it possible to reach the iSCSI targets, Ollama API, or Sunshine streaming endpoint from outside the LAN without any port forwarding through the D5001.

One interesting detail from the audit is that the iSCSI service is listening on the Tailscale IP (100.95.230.4) as well as the LAN IP. This means, at least in principle, an iSCSI initiator on another Tailscale node could mount a StarWind target remotely over the WireGuard tunnel. Whether this is intentional or just a consequence of StarWind binding to all interfaces is worth confirming. Mounting iSCSI over a WireGuard tunnel is possible but performance would be limited by tunnel overhead and latency.

---

## Network Shares

Five SMB shares are configured, four of which are hidden administrative shares that Windows creates by default and one that is explicitly user-visible.

| Share | Path | Visible |
|---|---|---|
| ADMIN$ | C:\WINDOWS | No |
| C$ | C:\ | No |
| D$ | D:\ | No |
| E$ | E:\ | No |
| E | E:\ | Yes |
| IPC$ | — | No |

The `E` share without the trailing dollar sign is a visible network share accessible to any authenticated user browsing the network. The administrative dollar-sign shares are accessible only to local administrators. The non-hidden E share is something to keep an eye on — the ACL configuration should reflect whatever the actual intent is for that volume.

---

## Windows Firewall State

The Domain firewall profile is enabled. The Private and Public profiles are both disabled.

This is a meaningful problem. The Domain profile applies when the machine believes it is on a domain network, but if that classification changes — which can happen when the network configuration changes, after certain updates, or for other environmental reasons — and the Domain profile stops applying, the machine is left with no firewall at all. Private and Public profiles being disabled is not a configuration that should persist. The fix is straightforward; it just has not been done.

There is a broader conversation here about what firewall rules the Domain profile actually contains. Even with the Domain profile enabled, the rules it enforces need to reflect the actual threat model — not just the defaults Windows set when the profile was created.

---

## StarWind Virtual SAN

### What It Does and Why

StarWind VSAN is the software that turns this machine's local storage into network-accessible block devices. It runs as a Windows service and presents iSCSI targets on port 3260. Other machines on the network — primarily josiahlab — can mount these targets as block devices and use them for VM disks, shared storage, or any other block-level workload.

The appeal of this setup is that it repurposes the 1 TB and 2 TB spinning disks that would otherwise just be local Windows storage into something the rest of the lab can use. josiahlab has a 238 GB NVMe drive that fills up quickly with VM disks and Docker overlay filesystems. Having a larger block storage pool available over iSCSI on the network means VM disks can live on the Windows machine's HDDs, keeping the NVMe free for OS and hot data.

### Service States

| Service | Status | Notes |
|---|---|---|
| StarWindService | Running | Core engine — hosts the iSCSI targets |
| StarWindClusterService | Running | Coordinates HA if there are multiple StarWind nodes |
| StarWindSMISAgent | Running | Exposes storage via WBEM/CIM for management tools |
| StarWindNVMeoF | Stopped | NVMe over Fabrics target — not configured or in use |

The NVMe-oF service being stopped is not a problem with the current setup. NVMe-oF would offer lower latency than iSCSI for block storage over the network, but it is more complex to configure and the performance difference is less significant on a local GbE network than it would be over a dedicated high-speed fabric. iSCSI over GbE is more than adequate for VM disk storage at homelab scale.

### Configuration Details

| Parameter | Value |
|---|---|
| Log level | 1 — errors only |
| Log rotate size | 100 MB |
| Log files kept | 20 |
| Update check interval | 7 days |
| Copy ID | D055479A-6CAD-4437-A9CD-0AE6BDE44833 |
| Last update check | 2026-03-14 |
| VAAI Extended Copy | Enabled |
| VAAI Compare-and-Write | Enabled |
| VAAI Write Same | Enabled |
| ODX | Disabled |
| Windows Update service control | Enabled |

VAAI (vStorage APIs for Array Integration) features are enabled. These are VMware hardware offload hints — Extended Copy, Compare-and-Write, and Write Same allow a VMware hypervisor to offload certain operations to the storage array rather than the host. In this lab, josiahlab uses KVM rather than VMware, so the VAAI features have no practical effect. They are enabled by default and there is no reason to disable them.

Windows Update service control being enabled means StarWind has taken ownership of when Windows Updates are allowed to run, presumably to prevent a system restart from happening while iSCSI targets are actively in use. This is reasonable behaviour for a machine serving storage to other hosts.

---

## Ollama (Windows)

The Windows machine runs Ollama in addition to josiahlab running it on Linux. The motivation is the GPU. josiahlab has integrated Intel Tiger Lake graphics with no discrete GPU. The RTX 3080 in the Windows machine has 10 GB GDDR6X VRAM and full CUDA support, which makes it significantly faster for inference — especially for models that fit in VRAM and can run entirely on the GPU.

The practical split is: josiahlab handles small, low-latency inference requests that do not need GPU acceleration. The Windows machine handles larger models or anything where inference speed matters.

Currently Ollama is binding to `::` (all interfaces) on Windows with no authentication. Any device on the LAN or Tailscale mesh can make inference requests to it. This is something that needs to be addressed — the RTX 3080 is a meaningful compute resource and there is no reason it should be freely accessible to anything on the network.

---

## Sunshine Game Streaming

Sunshine is running and listening on ports 47984, 47989, 47990, and 48010. This is the open-source NVIDIA GameStream replacement server. Paired with Moonlight clients on any supported device, it gives hardware-encoded (NVENC) desktop or game streaming at low latency. The RTX 3080's NVENC encoder handles compression, so streaming does not meaningfully burden the CPU.

With Tailscale active, Sunshine is accessible from outside the LAN without port forwarding. The quality of the stream over WireGuard depends heavily on the available bandwidth and latency of the connection, but for local streaming over GbE it is effectively lossless.

---

## PostgreSQL

PostgreSQL is running and binding to all interfaces (0.0.0.0 and ::). At the time of the audit it was not clear what application is using this PostgreSQL instance. It could be StarWind (some versions use PostgreSQL for the management database), or it could be something else entirely. The binding to all interfaces is the more pressing question — unless something external genuinely needs to connect to this database, it should be localhost-only.

---

## Open Ports

| Port | Protocol | Bind | Service |
|---|---|---|---|
| 22 | TCP | 0.0.0.0 / :: | OpenSSH |
| 135 | TCP | 0.0.0.0 / :: | RPC Endpoint Mapper |
| 139 | TCP | 192.168.0.37 | NetBIOS Session |
| 445 | TCP | :: | SMB |
| 3260 | TCP | 192.168.0.37, 100.95.230.4, 127.0.0.1 | iSCSI (StarWind) |
| 3261 | TCP | 0.0.0.0 | iSCSI secondary |
| 3333 | TCP | 127.0.0.1 | StarWind Management Console |
| 5432 | TCP | 0.0.0.0 / :: | PostgreSQL |
| 5988 | TCP | 0.0.0.0 / :: | WBEM / CIM (StarWind SMI-S) |
| 8811 | TCP | :: | Unidentified |
| 9000 | TCP | :: | Unidentified |
| 9991 | TCP | :: | Unidentified |
| 11434 | TCP | :: | Ollama |
| 12372 | TCP | :: | Unidentified |
| 27036 | TCP | 0.0.0.0 | Steam Remote Play |
| 27060 | TCP | 127.0.0.1 | Steam (local) |
| 47984-48010 | TCP | 0.0.0.0 | Sunshine game streaming |

Ports 8811, 9000, 9991, and 12372 are unidentified. They were not matched to any known service at audit time and need to be investigated.

---

## Storage Layout

| Drive | Estimated Size | Current Use |
|---|---|---|
| SSD | ~120 GB | Windows OS (C:\) |
| HDD 1 | ~1 TB | Data, StarWind virtual disk storage |
| HDD 2 | ~2 TB | Data, StarWind virtual disk storage |

---

## Future Plans

The most pressing hardware upgrade is RAM. Getting to dual-channel — at minimum adding one more matched DDR4 DIMM — would noticeably improve performance across the board, particularly for Ollama inference and parallel CPU workloads. The platform supports much more than 16 GB, so the upgrade path is wide open.

The Sunshine/Moonlight setup could be extended to cover more scenarios. Currently it is used for gaming and remote desktop access from within the Tailscale mesh. An interesting extension would be using it as a GPU compute endpoint — running Ollama inference requests on the RTX 3080 from any device on the Tailscale network, treating the Windows machine as a dedicated AI inference server that happens to also be a gaming PC.

The unknown ports (8811, 9000, 9991, 12372) need to be identified and documented. Running with unknown listeners on a machine that also serves iSCSI storage to other lab nodes is not ideal from a hygiene standpoint.

---

## Proof of Work

- [ ] StarWind Management Console — targets and connected initiator sessions
- [ ] Windows Firewall — all three profiles showing correct state
- [ ] Task Manager — GPU tab showing Ollama utilisation during inference
- [ ] Identify the four unknown port listeners
- [ ] Sunshine dashboard (https://localhost:47990) — configured apps and streaming settings
- [ ] PostgreSQL — identify which application owns this instance
