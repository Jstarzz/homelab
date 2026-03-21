# Security Audit

**Date:** 2026-03-20
**Scope:** DESKTOP-T6EKV7E, josiahlab, CT100

---

## Overview

The lab's security posture is typical for a homelab that has grown organically — reasonable in some areas, with specific gaps that have accumulated over time and not yet been addressed. Nothing is critically broken in a way that would immediately compromise the lab, but there are several issues that create more exposure than necessary, particularly given that Tailscale connects all of these machines into a mesh that also includes personal devices.

The most significant structural gap is the absence of a network-level firewall. The D5001 provides basic NAT (which prevents unsolicited inbound connections from the internet) but no configurable firewall ruleset. Host-based firewalls on each machine are the primary defence, and those are currently in a poor state: UFW is inactive on both Linux nodes, and two of three Windows Firewall profiles are disabled. The lab is effectively relying on the fact that the LAN is trusted, which is a reasonable assumption until something on the LAN is compromised.

---

## Risk Register

| Severity | Finding | Node |
|---|---|---|
| High | Windows Private and Public Firewall profiles disabled | DESKTOP-T6EKV7E |
| High | PostgreSQL (:5432) bound to 0.0.0.0 — accessible from entire LAN | DESKTOP-T6EKV7E |
| Medium | Ollama (:11434) on all interfaces with no authentication — josiahlab | josiahlab |
| Medium | Ollama (:11434) on all interfaces with no authentication — Windows | DESKTOP-T6EKV7E |
| Medium | OpenClaw CLI crashed and unrecovered — degraded application state | josiahlab |
| Medium | UFW inactive — INPUT chain accepts all traffic from LAN | josiahlab, CT100 |
| Medium | SMB share `E` (non-hidden) — ACL configuration unconfirmed | DESKTOP-T6EKV7E |
| Medium | Four unknown port listeners on Windows — unidentified attack surface | DESKTOP-T6EKV7E |
| Medium | josiah-api on :9090 — undocumented, auth status unknown | josiahlab |
| Low | No edge firewall between D5001 and LAN | Network |
| Low | Indirect Target Selection unmitigated on i3-1115G4 | josiahlab |
| Low | Multiple unmitigated CPU vulnerabilities on PVE host i3-7100U | CT100/PVE |
| Low | iSCSI CHAP authentication — not confirmed as configured | DESKTOP-T6EKV7E |
| Low | Prometheus and Node Exporter have no authentication | CT100 |

---

## No Edge Firewall

The network path is ISP → D5001 → GS108E switch → all hosts. There is no pfSense, OPNsense, or equivalent between the modem and the rest of the network. The D5001 provides NAT and stateful packet inspection for inbound internet traffic, but it is not user-configurable as a proper firewall. There are no ACL rules, no IDS/IPS, no per-VLAN policy.

In practice, the immediate risk is limited because:
1. No ports are forwarded through the D5001 to the internet — remote access is entirely via Tailscale's WireGuard tunnels
2. The LAN consists of known, trusted machines
3. The D5001's NAT prevents unsolicited inbound connections

The risk that does exist is internal — a compromised device on the LAN (or a new device that connects to the network without going through any vetting process) has unrestricted access to every service on every machine. This is the situation that VLANs and an edge firewall would address.

This is on the roadmap. The plan is to introduce a small x86 machine running OPNsense or pfSense between the D5001 and the GS108E. This would provide proper firewall rules, VLAN segmentation, DNS control, and IDS/IPS capability. The GS108E already supports 802.1Q VLANs, so the hardware is ready — it is just a matter of acquiring and configuring the firewall appliance.

---

## Windows Firewall Profiles

The Domain profile is active. The Private and Public profiles are both disabled.

This situation likely happened during troubleshooting at some point — disabling firewall profiles is a common step when diagnosing network connectivity problems — and the profiles were never re-enabled. The Domain profile applies when Windows believes it is on a domain network. If the network location classification changes or the Domain profile configuration is reset, the machine would be left with no firewall protection at all.

The fix is straightforward: enable both profiles. The more involved question is whether the Domain profile's rule set is actually appropriate for what this machine is doing. Default Domain profile rules may not account for the StarWind iSCSI service, Ollama, Sunshine, or any of the other non-standard services running here.

---

## UFW on Linux Nodes

Both josiahlab and CT100 have UFW installed but inactive. The iptables configuration on both machines is not empty — Docker and Tailscale both inject rules — but the INPUT chain accepts all traffic by default, with no explicit per-service rules.

What this means in practice: any device on the 192.168.0.0/24 subnet can reach SSH (port 22), Samba (445), Ollama (11434), CasaOS (80), AdGuard (53), Grafana (3000), Prometheus (9090), Portainer (9000), and every other listening port on both machines without restriction.

One important nuance is that UFW and Docker do not interact cleanly. Docker manages its own iptables chains (`DOCKER`, `DOCKER-FORWARD`) and these are in a higher-priority position than UFW's rules for forwarded traffic. Enabling UFW with `deny incoming` will protect the host's own services (SSH, Node Exporter, etc.) but Docker-published ports will remain accessible regardless — Docker's rules are applied in the `PREROUTING` chain before UFW's rules are evaluated. To restrict container-published ports, the compose file needs to bind container ports to specific host interfaces rather than relying on UFW.

For example, changing `ports: "3001:80"` to `ports: "127.0.0.1:3001:80"` in the AdGuard compose definition would restrict AdGuard's UI to localhost only, regardless of UFW state. This is the correct pattern for services that should only be accessible over Tailscale.

---

## Ollama — Open API Endpoints

Both Ollama instances are binding to all interfaces without authentication. This matters more on the Windows machine (GPU inference is a meaningful resource) than on josiahlab (CPU inference is less valuable). In the current threat model — trusted LAN, Tailscale mesh with personal devices — the practical risk is low. But if the Tailscale mesh ever includes a device that is itself compromised (a phone with malware, for instance), an open Ollama API is a free compute resource for whoever controls that device.

The intended end state is:
- josiahlab's Ollama binds to localhost only, serving as a local inference endpoint for services running on josiahlab itself
- Windows' Ollama binds to localhost only but is exposed selectively via a reverse proxy (Nginx or Caddy) with an API key requirement, accessible over Tailscale for legitimate use

---

## OpenClaw CLI Crash

The OpenClaw CLI container has been in an exited state for approximately 29 hours before the audit. This is an application-level crash — the gateway container is healthy, so it is specific to the CLI. The degraded state has persisted for over a day without being detected or addressed, which is a monitoring gap as much as an application problem.

If the monitoring stack on CT100 included a check for this container's health, an alert could have fired when the container exited. Setting up container health monitoring in Prometheus/Grafana (Docker container state as a metric) would catch this kind of failure automatically.

---

## PostgreSQL Exposure

PostgreSQL binding to all interfaces is the highest-priority item to address on the Windows machine. It is worse than the firewall issue in some ways because even with the Windows Firewall Private profile re-enabled, the default firewall rules might allow port 5432 through on the LAN. PostgreSQL authentication is managed by its own `pg_hba.conf` configuration, which may or may not be restrictive.

The first step is identifying which application owns this database. Until that is known, the appropriate change cannot be made safely — blindly restricting PostgreSQL to localhost might break something. Once the owning application is identified and its network requirements are understood, the access can be properly scoped.

---

## CPU Vulnerabilities

### josiahlab (i3-1115G4, Tiger Lake)

| Vulnerability | Status |
|---|---|
| Meltdown | Not affected |
| Spectre v1 | Mitigated |
| Spectre v2 | Mitigated |
| MDS | Not affected |
| Retbleed | Not affected |
| Indirect Target Selection | Vulnerable — no fix available |
| L1TF | Not affected |

The Indirect Target Selection vulnerability is the only outstanding issue on this CPU. It requires an attacker with local code execution to exploit. For a server without multi-tenant users, the practical risk is low.

### PVE host / CT100 (i3-7100U, Kaby Lake)

| Vulnerability | Status |
|---|---|
| Meltdown | Mitigated (PTI) |
| Spectre v1 | Mitigated |
| Spectre v2 | Mitigated |
| MDS | Vulnerable — no microcode |
| Retbleed | Vulnerable |
| Spec Store Bypass | Vulnerable |
| SRBDS | Vulnerable — no microcode |

The i3-7100U's vulnerability profile is worse because it is older and Intel stopped providing microcode updates for it. Since all containers on the Proxmox host share the host kernel, these vulnerabilities are shared by CT100 and any other containers or VMs on the node. There is no fix. The practical implication is that this hardware is not suitable for running untrusted code in containers — it should only run trusted workloads.

---

## Service Exposure Summary

| Service | LAN-accessible | Tailscale-accessible | Authentication |
|---|---|---|---|
| SSH (josiahlab) | Yes | Yes | Key or password |
| CasaOS | Yes | Yes | CasaOS login |
| AdGuard Home | Yes | Yes | Admin login |
| DNS (:53) | Yes | Yes | Open — no auth on DNS |
| Ollama (josiahlab) | Yes | Yes | None |
| OpenClaw | Yes | Yes | Unknown |
| josiah-api | Yes | Yes | Unknown |
| Grafana | Yes | Yes | Login required |
| Prometheus | Yes | Yes | None |
| Portainer | Yes | Yes | Login required |
| Node Exporter | Yes | Yes | None |
| iSCSI | Yes | Yes | CHAP — unconfirmed |
| Sunshine | Yes | Yes | PIN pairing |
| Ollama (Windows) | Yes | Yes | None |
| PostgreSQL | Yes | Yes | pg_hba rules — unconfirmed |

Internet exposure: no services appear to be directly port-forwarded through the D5001. All remote access is through Tailscale. This should be confirmed by reviewing the D5001's NAT/port forwarding table.

---

## Remediation Priority

The following captures what needs to happen, roughly in order of priority:

1. Re-enable Windows Firewall Private and Public profiles — quick fix, should be done immediately.

2. Identify the PostgreSQL owner, then restrict it to localhost or the specific IP that needs access.

3. Restrict Ollama to localhost on both machines, then decide on a proper access pattern for remote inference requests.

4. Enable UFW on josiahlab and CT100. Remember that UFW alone does not restrict Docker-published ports — container port bindings need to be updated alongside the firewall rules.

5. Investigate and fix the OpenClaw CLI crash. Add container health monitoring to catch this kind of failure automatically going forward.

6. Confirm that iSCSI CHAP authentication is configured in StarWind for all targets, so that an initiator without credentials cannot mount the storage.

7. Identify the four unknown port listeners on the Windows machine (8811, 9000, 9991, 12372).

8. Review and confirm the ACL on the non-hidden `E` SMB share.

9. Document josiah-api and determine whether it needs authentication.

10. Longer term: introduce an edge firewall appliance, implement VLANs, and move toward a properly segmented network.
