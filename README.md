# Homelab
[![LinkedIn](https://img.shields.io/badge/LinkedIn-TJ%20Banegas-0077B5?style=flat&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/tjbanegas)

A self-hosted homelab running on Proxmox VE. The two main goals are replacing big-tech cloud services with self-hosted alternatives, and building real infrastructure skills I can use professionally.

Everything here is documented on purpose. That includes the design decisions, the tradeoffs, and the stuff that broke. The troubleshooting is just as valuable as the end result.

**Career direction:** IT/Systems Administration → Cloud Security / DevSecOps / GRC

---

## Architecture

```
┌─────────────────────────────────────────────────────┐
│                   HP Z2 Mini G5                     │
│                  Proxmox VE 8.x                     │
│                                                     │
│  ┌─────────────────────┐  ┌──────────────────────┐  │
│  │   Ubuntu 24.04 VM   │  │   Ubuntu 25.04 LXC   │  │
│  │    (docker-host)    │  │      (pihole)        │  │
│  │                     │  │                      │  │
│  │  Docker Engine      │  │  Pi-hole v6          │  │
│  │  Portainer CE       │  │  Unbound             │  │
│  │  Uptime Kuma        │  │  Tailscale           │  │
│  │  Signal CLI API     │  │  Netdata (child)     │  │
│  │  Netdata (parent)   │  └──────────────────────┘  │
│  └─────────────────────┘                            │
│                                                     │
│  Netdata (child) · NUT · Tailscale                  │
└─────────────────────────────────────────────────────┘
                        │
              Tailscale Overlay Network
                        │
      ┌──────────────────────────────────────┐
      │        Ubuntu 24.04 VM               │
      │     Ansible Control Node             │
      │   (ubuntu-ansiblehost)               │
      └──────────────────────────────────────┘
```

All remote access goes through Tailscale. No ports are forwarded from the internet to any host.

---

## Stack

| Category | Technology | Notes |
|---|---|---|
| Hypervisor | Proxmox VE 8.x | Type-1 hypervisor; runs VMs and LXC containers |
| DNS | Pi-hole v6 + Unbound | Ad blocking + recursive DNS from root servers |
| Containers | Docker Engine + Portainer CE | Container runtime with a web UI |
| Monitoring | Netdata v2 (parent/child streaming) | Centralized metrics; children stream to parent |
| Alerting | Uptime Kuma + Signal CLI REST API | End-to-end encrypted notifications, no third-party relay |
| Automation | Ansible + ansible-vault | Reproducible deployments with secrets encrypted at rest |
| Networking | Tailscale (MagicDNS, ACLs) | Zero-trust overlay network |
| Power | NUT (Network UPS Tools) | Graceful shutdown on UPS battery low |
| OS | Ubuntu 24.04 LTS, Ubuntu 25.04, Debian 12 | Mixed across VMs and containers |
| Firewall | UFW | Default-deny on all Ubuntu hosts |

---

## Core Services

### DNS: Pi-hole + Unbound

Pi-hole handles network-wide DNS filtering and ad blocking. Unbound sits behind it as a recursive resolver, so DNS queries go straight to root servers instead of passing through Google (8.8.8.8), Cloudflare (1.1.1.1), or my ISP.

**Why co-locate them in one LXC?** Running both in the same container means Pi-hole talks to Unbound over localhost (port 5335) instead of across the network. It's simpler, lower overhead, and there's no reason to expose Unbound on a network-accessible port.

**Why not just use a public DNS provider?** Upstream resolvers log your queries and tie them to your IP. Recursive resolution means no single provider sees your full DNS history.

---

### Monitoring: Netdata Parent/Child Streaming

Netdata runs in a parent/child setup:

- **Parent** (docker-host): collects and stores metrics from all nodes, serves the unified dashboard. Storage is tiered: full resolution for 24 hours, 1-minute resolution for 30 days, 1-hour resolution for about a year.
- **Children** (proxmox, pihole): collect metrics locally and stream them to the parent. Local storage is disabled (`memory mode = none`), which cuts per-host RAM usage by around 70%.

**Why not use Netdata Cloud?** Netdata Cloud routes your metrics through Netdata's own infrastructure. Keeping everything on-prem means no third party has visibility into my system performance data.

**Bugs I hit in Netdata v2 (none of these were in the docs):**
- The parent needs a `[stream] enabled = no` global section in stream.conf or the streaming receiver never starts, even if the API key config looks correct
- CIDR notation in `allow from` (like `192.168.0.0/16`) silently fails; you need a wildcard `*` or a specific IP
- Static builds on Ubuntu 25.04 install to `/opt/netdata/etc/netdata/`, not `/etc/netdata/`; the config path has to be detected dynamically
- Each child needs its own unique API key; shared keys cause silent rejections with no useful error output
- Ubuntu 25.04 requires `--install-type static` because the distro isn't in the native package repo yet

All of this is now handled in the Ansible role.

---

### Alerting: Uptime Kuma + Signal

Uptime Kuma monitors the stack with checks that actually test functionality, not just connectivity:

- Proxmox web UI: HTTPS keyword check (confirms the login page loads, not just that the port is open)
- Pi-hole DNS: sends a real DNS query to the resolver; catches pihole-FTL crashes that a ping check would miss
- Docker containers: monitors container state via the Docker socket
- Signal CLI API: container health check

Alerts go out through a self-hosted `signal-cli-rest-api` container. It's linked as a secondary device on an existing Signal account, so no new phone number is needed and nothing routes through a third-party relay.

**Known limitation:** Kuma and the Signal API both run on docker-host. If that host goes down, alerts stop working. The fix is an external heartbeat monitor (like Healthchecks.io) that pages out if it stops hearing from Kuma. That's on the roadmap.

---

### Automation: Ansible Role

Ansible manages Netdata deployment across all hosts.

**What the role does:**
- Installs Netdata via the official kickstart script with an idempotency check (skips reinstall if the binary already exists)
- Detects the config directory dynamically to handle both native (`/etc/netdata/`) and static (`/opt/netdata/etc/netdata/`) installs
- Templates `stream.conf` and `netdata.conf` per host using Jinja2
- Encrypts API keys with ansible-vault; no plaintext secrets ever hit git
- Passes `--install-type static` for Ubuntu 25.04
- Uses handlers so Netdata only restarts when config actually changes

**Secrets:** The vault password lives in `~/.vault_pass` on the control node only and is excluded from git via `.gitignore`. The repo is safe to make public.

---

### Power Management: NUT

A CyberPower 750VA UPS is connected to the Proxmox host via USB. NUT watches battery state and triggers a graceful hypervisor shutdown before the battery runs out, which prevents filesystem corruption from a hard power cut.

Configured in standalone mode. Verified working: `ups.status: OL` confirmed via `upsc`.

---

## Security Hardening

The threat model here is a home network with no internet-facing services. That said, hardening is applied as if the LAN could be compromised, because a vulnerable IoT device or misconfigured guest could realistically get there.

### Network

- No ports are forwarded from the internet. All remote access goes through Tailscale.
- Tailscale ACLs are scoped by device role:
  - `tag:admin` (laptops, control node): full access to everything
  - `tag:server` (VMs): open inter-server access for Netdata streaming, Ansible SSH, and DNS
  - `tag:personal` (phone): web UIs only (ports 80, 443, 3001, 9443, 19999, 8006), no SSH
  - Entertainment devices (Apple TV): no infrastructure access

### Host Hardening (all managed hosts)

- Root SSH login disabled
- Password auth disabled; SSH key authentication only
- UFW on all Ubuntu hosts, default-deny with explicit allow rules for LAN (`192.168.1.0/24`) and Tailscale (`100.64.0.0/10`)
- Unattended-upgrades running on all Ubuntu hosts

### Docker

- Docker's built-in iptables management is disabled (`{"iptables": false}` in `daemon.json`). By default, Docker bypasses UFW and exposes container ports regardless of your firewall rules. This fixes that.
- Signal CLI API is not reachable externally; Kuma talks to it over the LAN IP only.

### Proxmox

- TOTP 2FA on the web UI

### Secrets

- Ansible-vault for all sensitive values in the repo
- SSH keys never leave their originating host
- Unique passwords per service; no shared credentials

---

## Repository Structure

```
homelab/
├── ansible/
│   ├── ansible.cfg
│   ├── site.yml
│   ├── inventory/
│   │   └── hosts.yml
│   ├── group_vars/
│   │   └── all/
│   │       ├── vars.yml
│   │       └── vault.yml
│   └── roles/
│       └── netdata/
│           ├── defaults/main.yml
│           ├── tasks/
│           │   ├── main.yml
│           │   ├── install.yml
│           │   ├── parent.yml
│           │   └── child.yml
│           ├── templates/
│           │   ├── stream-parent.conf.j2
│           │   ├── stream-child.conf.j2
│           │   ├── netdata-parent.conf.j2
│           │   └── netdata-child.conf.j2
│           └── handlers/main.yml
├── monitoring/
├── networking/
├── infrastructure/
└── docs/
```

---

## What I Would Do Differently

- **Set up Netdata manually before touching Ansible.** I tried to automate it before I fully understood how the streaming config worked. That meant debugging the service and the automation at the same time, which made both harder. The right sequence is: get it working by hand, then automate it.
- **Use unique API keys from the start.** I started with a shared key across all children. Netdata v2 silently rejects duplicate connections, so it just didn't work with no useful error. Should have been unique per child from day one.
- **Disable Docker's iptables before enabling UFW.** I turned on UFW first and assumed my firewall rules were working. They weren't. Docker had already punched holes through iptables and was exposing container ports. The correct order: set `daemon.json` first, restart Docker, then enable UFW.
- **Use cloud-init from the beginning.** My first VMs used the interactive installer, which has a known Shift key bug in Proxmox's noVNC console. Cloud-init images skip the console entirely and are faster and more reproducible anyway.

---

## Roadmap

- [ ] pfSense with VLAN segmentation (blocked on getting a managed switch that supports 802.1Q trunking)
- [ ] TrueNAS Scale on UGREEN DXP4800 Plus; RAID-Z2 across 4x10TB, 32GB RAM
- [ ] 3-2-1 backup strategy: TrueNAS as primary, Synology off-site, cold cloud storage
- [ ] Immich for self-hosted photos (replacing Google Photos)
- [ ] Nextcloud for file storage
- [ ] Caddy as a reverse proxy for internal TLS termination
- [ ] Prometheus + Grafana for a proper observability stack
- [ ] Wazuh SIEM for security event monitoring
- [ ] Azure projects: Sentinel, Defender for Cloud, Terraform/Bicep IaC
- [ ] Complete NUT live failover test
- [ ] Ansible roles for remaining services

---

## Hardware

| Component | Device |
|---|---|
| Hypervisor host | HP Z2 Mini G5 |
| NAS (incoming) | UGREEN DXP4800 Plus |
| UPS | CyberPower CP750SL 750VA Slimline |
| Network | Consumer router (pfSense replacement planned) |
