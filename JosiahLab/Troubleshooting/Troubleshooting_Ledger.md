# Troubleshooting Ledger

A running log of known issues, their root causes, and how they were resolved (or the current state of investigation). Issues are added here when they are identified and updated when they are fixed.

---

## Issue #001 — OpenClaw CLI Container Exited with Code 1

| Field | Detail |
|---|---|
| Node | josiahlab |
| First observed | ~2026-03-19 (~29 hours before audit) |
| Current state | Unresolved — container remains exited |

### What happened

The `openclaw-openclaw-cli-1` container exited with code 1 approximately 29 hours before the March 2026 audit. The gateway container (`openclaw-openclaw-gateway-1`) is running and healthy. Docker's restart policy did not bring the CLI back up — either the restart policy is `no` or `on-failure` with a max attempt limit that was exhausted.

### Investigation starting points

Exit code 1 is a generic application-level error rather than a signal or OOM condition. It means the application itself exited with a non-zero status — a deliberate exit after encountering an error it could not recover from. Since the gateway is healthy and the CLI shares the same Docker network (`openclaw_default`), network connectivity between them is not the problem.

The most likely causes, in rough order of probability:
- A missing or incorrect environment variable or configuration file that the CLI requires on startup
- An authentication failure when the CLI tries to connect to the gateway API
- A race condition where the CLI started before the gateway was fully initialised and ready to accept connections (this would be a startup ordering issue that only manifests on cold starts or restarts)
- An application bug introduced in the CLI image build

```bash
# Primary diagnostic: read the container exit logs
docker logs openclaw-openclaw-cli-1

# With timestamps for correlation
docker logs --timestamps --tail 200 openclaw-openclaw-cli-1

# Check restart policy and exit state
docker inspect openclaw-openclaw-cli-1 | grep -A10 '"State"'
docker inspect openclaw-openclaw-cli-1 | grep -A5 '"RestartPolicy"'

# Look at the compose file for environment variables and depends_on
find / -name "docker-compose.yml" -path "*/openclaw/*" 2>/dev/null
```

### Resolution

Dependent on what the logs show. If it is a race condition, adding a `depends_on` healthcheck condition in the compose file (waiting for the gateway's healthcheck to pass before starting the CLI) would fix it. If it is a missing config, finding and supplying the correct value. If it is a bug in the local image build, rebuilding the image.

Once fixed, adding a Prometheus/Grafana alert for container exit states would prevent this kind of failure from going undetected for 29 hours again.

---

## Issue #002 — josiahlab RAM Under Constant Pressure

| Field | Detail |
|---|---|
| Node | josiahlab |
| First observed | 2026-03-20 audit |
| Current state | Ongoing — hardware constraint |

### The situation

At audit time: 5.8 GiB of 7.5 GiB RAM used, 1.9 GiB of 4 GiB swap in use, 345 MiB of free RAM. The machine is running: a KVM virtual machine, the AdGuard Home container, the OpenClaw gateway container, the Tailscale container, CasaOS and its microservices, libvirtd, Ollama (with whatever model is currently loaded), Samba, rclone, iperf3, and josiah-api. That is a lot of services for 8 GB.

### Root cause

A single 8 GB DDR4-3200 SODIMM occupying slot 1, with slot 2 empty. This is both a capacity constraint (8 GB total) and a bandwidth constraint (single-channel mode cuts memory throughput roughly in half). Ollama is the biggest contributor to the pressure — holding a loaded model in RAM is expensive, and the model stays resident even when not actively inferring.

### Workarounds

Stopping Ollama when it is not needed is the most impactful short-term action. A loaded 7B parameter model at Q4 quantisation takes roughly 4 GB of RAM. Freeing that back to the system when inference is not happening would drop swap usage significantly.

```bash
# Stop Ollama to reclaim memory
sudo systemctl stop ollama

# Check what processes are using the most memory right now
ps aux --sort=-%mem | head -20

# Check swap usage per process (requires smem)
smem -rs swap | head -20
```

### Fix

Adding a second 8 GB DDR4-3200 SODIMM to the empty slot 2. This is the correct and complete solution — it doubles capacity to 16 GB and enables dual-channel operation, both of which help. The Dell Latitude 3520 supports up to 32 GB (2× SODIMM, DDR4-3200, 1.2V, non-ECC, 260-pin).

A 16 GB machine would be able to comfortably run the KVM VM, all the containers, and Ollama with a 7B model loaded simultaneously without swapping.

---

## Issue #003 — Windows Firewall Private and Public Profiles Disabled

| Field | Detail |
|---|---|
| Node | DESKTOP-T6EKV7E |
| First observed | 2026-03-20 audit |
| Current state | Unresolved |

### What happened

Both the Private and Public Windows Firewall profiles are disabled. The Domain profile is enabled. The most likely explanation is that the profiles were disabled during a troubleshooting session — network connectivity problems are sometimes diagnosed by disabling the firewall, and the fix is not always reversed. It is also possible a software installer or configuration script disabled them.

### Why it matters

The Domain profile applying depends on Windows correctly classifying the current network connection as a domain network. If that classification changes — for any reason — the machine loses all firewall protection. Even with the Domain profile working, the actual rules it contains may not reflect the current set of services running on the machine.

### Resolution

```powershell
Set-NetFirewallProfile -Profile Private,Public -Enabled True
```

After re-enabling, the Domain profile rules should also be reviewed to confirm they actually cover the services running (StarWind, Ollama, Sunshine, OpenSSH, PostgreSQL, etc.) rather than just whatever Windows configured by default.

---

## Issue #004 — PostgreSQL Bound to All Interfaces

| Field | Detail |
|---|---|
| Node | DESKTOP-T6EKV7E |
| First observed | 2026-03-20 audit |
| Current state | Unresolved |

### What happened

PostgreSQL is running with `listen_addresses` set to the default, which binds to all interfaces. This is a common default-configuration issue — the installer does not make the service localhost-only unless specifically configured.

### What needs to happen first

The owning application needs to be identified before changing the bind address. Changing it to localhost and then discovering that some service was connecting remotely would cause an outage. The investigation looks like:

```powershell
# Find which process owns the PostgreSQL service
Get-WmiObject Win32_Service | Where-Object {$_.Name -like "postgresql*"} | Select-Object Name, PathName, StartName

# Check pg_hba.conf for remote authentication rules
# (location varies by version)
Get-ChildItem "C:\Program Files\PostgreSQL" -Recurse -Filter pg_hba.conf

# Check if anything is currently connected remotely
# Run in psql:
# SELECT client_addr, application_name, state FROM pg_stat_activity WHERE client_addr IS NOT NULL;
```

Once the owning application is known and its network requirements are understood, `listen_addresses = 'localhost'` can be set safely in `postgresql.conf` with a service restart.

---

## Issue #005 — UFW Inactive on Linux Nodes

| Field | Detail |
|---|---|
| Nodes | josiahlab, CT100 |
| First observed | 2026-03-20 audit |
| Current state | Unresolved |

### Background

UFW was never configured on either node. Docker and Tailscale inject their own iptables rules, but the host's INPUT chain defaults to ACCEPT — there are no rules preventing arbitrary LAN devices from reaching SSH, Samba, Ollama, or any other host-level service.

### The Docker complication

Docker bypasses UFW for forwarded ports. Docker's DNAT rules are in the PREROUTING chain, which is evaluated before UFW's filter rules. Enabling `ufw default deny incoming` will protect host services but will not block traffic to container-published ports. This is a well-known Docker/UFW interaction problem and the correct solution is to bind container ports to specific interfaces in the compose file rather than relying on UFW.

For example, the AdGuard compose currently publishes `:53` which binds to all host interfaces. Changing this to `192.168.0.28:53:53` would restrict DNS to the LAN interface only, and no UFW rule is needed for that. This approach — tightening port bindings in compose files — should be done alongside enabling UFW.

### Planned configuration — josiahlab

```bash
ufw default deny incoming
ufw default allow outgoing
ufw allow from 192.168.0.0/24 to any port 22    # SSH from LAN
ufw allow from 192.168.0.0/24 to any port 445   # Samba
ufw allow from 192.168.0.0/24 to any port 139   # NetBIOS
ufw allow from 192.168.0.0/24 to any port 80    # CasaOS
ufw allow from 192.168.0.0/24 to any port 53    # DNS
ufw allow from 100.64.0.0/10 to any port 11434  # Ollama — Tailscale only
ufw allow in on tailscale0                        # All Tailscale traffic
ufw enable
```

### Planned configuration — CT100

```bash
ufw default deny incoming
ufw default allow outgoing
ufw allow from 192.168.0.0/24    # Full LAN access to CT100 services
ufw allow from 100.64.0.0/10     # Tailscale mesh
ufw allow in on tailscale0
ufw enable
```

---

## Issue #006 — josiahlab Single-Channel RAM

| Field | Detail |
|---|---|
| Node | josiahlab |
| First observed | 2026-03-20 audit |
| Current state | Pending hardware purchase |

### Summary

DIMM slot 2 is empty. This is a performance issue rather than a stability issue — the machine works, it is just slower at memory-intensive operations than it would be with two DIMMs installed. The fix is purchasing a second 8 GB DDR4-3200 SODIMM.

Required spec: DDR4 SODIMM, 8 GB, 3200 MHz (PC4-25600), 1.2V, non-ECC, 260-pin. A matching module from the same manufacturer as the existing stick (Micron, based on the manufacturer ID in the dmidecode output) would be ideal for dual-channel compatibility, though most DDR4 SODIMMs will work together.

After installing: `dmidecode -t memory` should show both DIMM 1 and DIMM 2 populated at 8192 MB and 3200 MT/s. The system should boot into 16 GB dual-channel mode. Memory pressure and swap usage should drop significantly.

---

## Issue #007 — josiah-api Undocumented

| Field | Detail |
|---|---|
| Node | josiahlab |
| First observed | 2026-03-20 audit |
| Current state | Needs investigation |

### What is known

A process named `josiah-api` is running and listening on port 9090 across all interfaces. It has been running long enough to appear in the audit as an active listener. Port 9090 is conventionally associated with Prometheus, but Prometheus is running on CT100. The name suggests it is a custom-built application rather than an off-the-shelf tool.

### What needs to be found out

- What does it do? Is it an API for some lab automation, a metrics endpoint, a proxy, or something else?
- Where is its source code? Is it version controlled?
- How is it started? Is there a systemd unit file for it?
- Does it have authentication? Given that it is on all interfaces, anything that connects to it should require auth.
- Does anything depend on it?

```bash
# Find the service definition
systemctl status josiah-api 2>/dev/null
cat /etc/systemd/system/josiah-api.service 2>/dev/null

# Find the binary
which josiah-api 2>/dev/null
ls -la /usr/local/bin/josiah-api 2>/dev/null
ls -la /opt/josiah-api 2>/dev/null

# Check what it is
file $(which josiah-api 2>/dev/null)
strings $(which josiah-api 2>/dev/null) | head -50
```

---

## Recurring Maintenance Reference

| Task | How | Frequency |
|---|---|---|
| Update Docker images on josiahlab | Per stack: `docker compose pull && docker compose up -d` | Monthly |
| Update monitoring stack on CT100 | `cd /opt/monitoring && docker compose pull && docker compose up -d` | Monthly |
| NVMe health check | `smartctl -a /dev/nvme0n1` on josiahlab | Monthly |
| Review AdGuard query stats | http://192.168.0.28:3001 | Weekly |
| Check Prometheus targets | http://192.168.0.26:9090/targets | Weekly |
| Review StarWind logs | StarWind Management Console on Windows | Weekly |
| Check Tailscale node expiry | `tailscale status` on any node | Monthly |
| Review Grafana dashboards | http://192.168.0.26:3000 | Weekly |
| Backup Portainer config | `docker run --rm -v portainer_data:/data busybox tar czf - /data > portainer_backup_$(date +%Y%m%d).tar.gz` | Monthly |
| Check rclone sync status | `systemctl status rclone` on josiahlab | Weekly |
| Check swap usage on josiahlab | `free -h` | Weekly until RAM is upgraded |
