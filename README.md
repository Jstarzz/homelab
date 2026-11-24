# homelab

Notes for the machines I run: two Proxmox servers, a home router, and the home lab documented in the Obsidian vault under [`JosiahLab/`](JosiahLab/000_HOME.md).

Inventory below was taken from the hosts on 2026-11-24. The vault is a March 2026 audit of the home lab and has not been re-audited since; treat its service and security tables as historical. Its PVE node (i3-7100U, CT100 `services`) is a different machine from `pve` below.

## Hosts

| Host | Hardware | OS | Role |
| --- | --- | --- | --- |
| `pve` | Xeon X5650, 24 threads, 41 GB RAM, 136 GB + 5.3 TB storage | Proxmox VE 9.1.1, kernel 6.17.2 | Main hypervisor. Reached through `bughuntinc`. |
| `bughuntinc` | Xeon X5670, 24 threads, 23 GB RAM | Proxmox VE 9.2 on Debian 13 | Client-work server and jump host for `pve`. |
| `router` | TP-Link Archer A7 v5 | OpenWrt 25.12.5 | Router. |
| `cubi` | Not audited | Not audited | On another tailnet; not reachable from the work tailnet during the audit. |
| `josiahlab`, `services`, desktop | See vault | See vault | Home lab on another tailnet. Not re-audited in November. |

## `pve` guests

| ID | Type | Name | State | Size | Purpose |
| --- | --- | --- | --- | --- | --- |
| 101 | CT | debian-ct | stopped | 2 GB | Spare Debian container |
| 106 | CT | geotagger-lite | running, on boot | 1 core, 1 GB | Geotagger behind a Cloudflare tunnel |
| 107 | CT | claude | running, on boot | 4 cores, 8 GB | Claude Code workspace |
| 110 | CT | ndhis-ai | running | 8 cores, 16 GB, 240 GB | NDHIS AI services |
| 111 | CT | web-scraper | running, on boot | 4 cores, 8 GB | [web-scraper](https://github.com/Jstarzz/web-scraper): API, workers, browser worker, Postgres, Cloudflare tunnel |
| 100 | VM | geotagger-prod | stopped | 9 cores, 16 GB | Full geotagger deployment |
| 102 | VM | ambulance-tracking | running | 4 cores, 4 GB | Ambulance tracking |
| 108 | VM | test-vm | running | 8 cores, 16 GB | Scratch VM |
| 210 | VM | ndhis-sandbox | running | 8 cores, 16 GB, 150 GB | NDHIS load and memory testing, synthetic data only |
| 420 | VM | debian-12-cloud | stopped | 2 GB | Cloud-init template |

Storage: `local` (130 GB, OS and small guests) and `bigstorage` (5.3 TB directory store, most guest disks).

## Tailscale

Three tailnets, kept separate by purpose:

| Tailnet | Use |
| --- | --- |
| bughuntinc | Client work. Most day-to-day access, including `bughuntinc`. |
| josiahdavis321 | Personal. |
| josiahdavis.ai | Gaming. |
