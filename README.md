# homelab
# Homelab Infrastructure

A self-hosted homelab running on Proxmox VE, built with two explicit goals: replacing
commercial cloud services with privacy-respecting self-hosted alternatives, and
developing hands-on infrastructure skills across Linux administration, networking,
security hardening, observability, and automation.

This repository documents the infrastructure as code, configuration, and design
decisions behind the lab. Every decision — including the tradeoffs and the things
that broke — is documented here intentionally. The troubleshooting and reasoning
are as important as the working result.

**Current trajectory:** IT/Systems Administration → Cloud Security / DevSecOps / GRC

---

## Architecture
─────────────────────────────────────────────────────┐
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

All remote access routes exclusively through Tailscale.
No ports are forwarded from the internet to any host.

---

## Stack

| Category | Technology | Notes |
|---|---|---|
| Hypervisor | Proxmox VE 8.x | Type-1 hypervisor; VMs and LXC containers |
| DNS | Pi-hole v6 + Unbound | Ad blocking + recursive resolution from root servers |
| Containers | Docker Engine + Portainer CE | Container runtime with web management UI |
| Monitoring | Netdata v2 (parent/child streaming) | Centralized metrics; children stream to parent |
| Alerting | Uptime Kuma + Signal CLI REST API | E2E encrypted notifications, no third-party relay |
| Automation | Ansible + ansible-vault | IaC for reproducible deployments; secrets encrypted at rest |
| Networking | Tailscale (MagicDNS, ACLs) | Zero-trust overlay; replaces VPN |
| Power | NUT (Network UPS Tools) | Graceful shutdown on UPS battery low |
| OS | Ubuntu 24.04 LTS, Ubuntu 25.04, Debian 12 | Mixed environment across VMs and containers |
| Firewall | UFW | Default-deny on all Ubuntu hosts |

---

## Core Services

### DNS: Pi-hole + Unbound

Pi-hole handles network-wide DNS filtering and ad blocking. Unbound runs as a
recursive resolver, eliminating dependency on upstream DNS providers. DNS queries
resolve directly from root servers rather than passing through Google (8.8.8.8),
Cloudflare (1.1.1.1), or the ISP's resolver.

**Design decision:** Pi-hole and Unbound are co-located in a single LXC container.
Unbound listens on `127.0.0.1:5335`; Pi-hole forwards upstream queries to it. This
avoids the overhead of container-to-container networking for what is effectively a
local IPC call, and reduces the attack surface compared to running Unbound on an
exposed port.

**Privacy rationale:** Upstream DNS providers log queries and associate them with IP
addresses. Recursive resolution means no single provider sees the full query history.

---

### Monitoring: Netdata Parent/Child Streaming

Netdata deployed in a parent/child streaming architecture:

- **Parent** (docker-host): receives and stores metrics from all nodes, serves the
  unified dashboard. Configured with tiered dbengine storage: full resolution for
  24h, 1-minute resolution for 30 days, 1-hour resolution for ~1 year.
- **Children** (proxmox, pihole): collect and stream metrics to parent.
  `memory mode = none` — no local storage, metrics ship directly to parent.
  Reduces per-host RAM footprint by approximately 70%.

**Design decision:** Self-hosted streaming over Netdata Cloud. Netdata Cloud routes
metrics through Netdata Inc's infrastructure. Keeping telemetry on-premises means
no third party has visibility into infrastructure performance data or potential
indicators of what services are running.

**Troubleshooting log (Netdata v2 undocumented behavior):**
These issues were found through debugging, not documentation:
- Parent requires a `[stream] enabled = no` global section in stream.conf or the
  streaming receiver does not initialize, even with valid API key sections present
- CIDR notation in `allow from` (e.g. `192.168.0.0/16`) silently fails in v2;
  wildcard `*` or specific IPs required
- Static builds on Ubuntu 25.04 install to `/opt/netdata/etc/netdata/` not
  `/etc/netdata/` — config path detection must be dynamic
- Each child requires a unique API key in v2; shared keys cause silent connection
  rejections with no useful error message
- `--install-type static` required for Ubuntu 25.04 as the native package
  repository does not yet carry the distro

These findings are codified in the Ansible role's dynamic config path detection
and per-host key assignment.

---

### Alerting: Uptime Kuma + Signal

Uptime Kuma monitors service health across the stack:
- Proxmox web UI: HTTPS keyword check (confirms login page is served, not just
  that the port is open)
- Pi-hole DNS: actual DNS query resolution against the Pi-hole resolver — detects
  pihole-FTL crashes that a ping monitor would miss
- Docker containers: Docker socket integration for container state monitoring
- Signal CLI API: Docker container health check

Alerts delivered via Signal through a self-hosted `signal-cli-rest-api` bridge.
The bridge is linked as a secondary device to an existing Signal account — no
new phone number required, no third-party relay, messages are end-to-end encrypted.

**Alerting design consideration:** both the monitoring system (Kuma) and the
notification path (Signal API) run on the same host. If that host goes down,
alerts cannot be delivered. This circular dependency is a known limitation for
homelab single-host setups. An external watchdog (e.g. Healthchecks.io with a
Kuma heartbeat monitor) closes this gap and is on the roadmap.

---

### Automation: Ansible Role

An Ansible role manages Netdata installation and configuration across all hosts.

**Role capabilities:**
- Installs Netdata via official kickstart script with idempotency guard (stat
  check on binary path prevents re-install on existing hosts)
- Detects config directory dynamically — handles both native installs
  (`/etc/netdata/`) and static builds (`/opt/netdata/etc/netdata/`)
- Templates `stream.conf` and `netdata.conf` per role assignment (parent or child)
  using Jinja2; `inventory_hostname` resolves correctly per host
- API keys encrypted with ansible-vault; plaintext secrets never committed to git
- `--install-type static` flag handles Ubuntu 25.04 package gap
- Handler-based restarts: Netdata only restarts when config actually changes,
  not on every playbook run

**Secrets management:** ansible-vault encrypts the API key at rest. The vault
password lives in `~/.vault_pass` on the control node only, excluded from git
via `.gitignore`. The repository is safe to make public.

---

### Power Management: NUT

CyberPower 750VA Slimline UPS connected to the Proxmox host via USB. NUT monitors
battery state and triggers a graceful shutdown of the hypervisor before battery
exhaustion, preventing filesystem corruption from hard power loss.

Configured in standalone mode on Proxmox. UPS confirmed reachable via `lsusb`
and `upsc` before software configuration. Verified `ups.status: OL` (on line).

---

## Security Hardening

Security is applied as defense-in-depth across the stack. The threat model is a
home network — no internet-facing services — but hardening is applied as if the
network boundary could be crossed, because it sometimes can be (compromised LAN
device, vulnerable IoT device, etc.).

### Network Perimeter

- **No ports forwarded from the internet.** All remote access exclusively through
  Tailscale. The attack surface visible from the internet is zero.
- **Tailscale ACLs** segment the tailnet by device role:
  - `tag:admin` (laptops, control node): full access to all infrastructure
  - `tag:server` (VMs): unrestricted inter-server communication for Netdata
    streaming, Ansible SSH, DNS
  - `tag:personal` (phone): web UI access only (ports 80, 443, 3001, 9443, 19999,
    8006) — no SSH
  - Entertainment devices (Apple TV): no access to infrastructure

### Host Hardening (all managed hosts)

- Root SSH login disabled (`PermitRootLogin no`)
- Password authentication disabled (`PasswordAuthentication no`) — SSH key
  authentication only
- Non-root admin users with scoped sudo (Ansible uses `NOPASSWD:ALL` for
  automation; this is a documented tradeoff acceptable for a single-operator lab)
- UFW firewall active on all Ubuntu hosts:
  - Default policy: deny incoming, allow outgoing
  - Allowed inbound: LAN subnet (`192.168.1.0/24`), Tailscale subnet
    (`100.64.0.0/10`)
- Unattended-upgrades configured for automatic security patching on all Ubuntu
  hosts

### Docker-Specific

- Docker's iptables management disabled (`{"iptables": false}` in `daemon.json`)
  to prevent Docker from bypassing UFW rules — a well-known default behavior that
  silently exposes container ports regardless of firewall state
- Signal CLI API container not exposed externally; accessed only from Kuma on the
  same host via LAN IP

### Proxmox

- TOTP two-factor authentication on web UI admin account
- NUT provides hardware-level protection against ungraceful shutdown

### Secrets

- Ansible-vault encrypts sensitive values at rest in the repository
- SSH private keys never leave their originating host
- All services use unique passwords; no shared credentials across services

---

## Repository Structure
homelab/
├── ansible/
│   ├── ansible.cfg              # Vault password path, inventory defaults
│   ├── site.yml                 # Main playbook
│   ├── inventory/
│   │   └── hosts.yml            # Host definitions, group assignments
│   ├── group_vars/
│   │   └── all/
│   │       ├── vars.yml         # Variable references (plaintext)
│   │       └── vault.yml        # Encrypted secrets (ansible-vault)
│   └── roles/
│       └── netdata/
│           ├── defaults/main.yml
│           ├── tasks/
│           │   ├── main.yml     # Validation and dispatch
│           │   ├── install.yml  # Idempotent install with path detection
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

---

## What I Would Do Differently

Documenting this is as useful as documenting what worked:

- **Netdata before Ansible, not instead of.** Setting up streaming manually first,
  then codifying it in Ansible, meant debugging both the service and the automation
  simultaneously. Manual setup → verify → automate is a cleaner sequence.
- **Separate API keys from the start.** Used a shared Netdata streaming API key
  across children initially. Netdata v2 silently rejects duplicate key connections.
  Per-child unique keys should be the default, not a fix.
- **UFW before Docker.** Enabling UFW on a Docker host without first disabling
  Docker's iptables management creates a false sense of security — Docker bypasses
  UFW silently. The correct order is: configure daemon.json → restart Docker →
  enable UFW.
- **Cloud-init for VM provisioning.** Initial VM setup used the interactive Ubuntu
  installer, which had persistent keyboard input issues in Proxmox's noVNC console.
  Cloud images with cloud-init are faster, more reproducible, and avoid the console
  entirely.

---

## Roadmap

- [ ] pfSense for network segmentation (blocked on managed switch for 802.1Q VLAN
  trunking via single NIC)
- [ ] TrueNAS Scale on UGREEN DXP4800 Plus (drives in transit); RAID-Z2 across
  4×10TB, 32GB RAM upgrade
- [ ] 3-2-1 backup strategy: primary on TrueNAS, off-site to Synology (incoming),
  cloud cold storage
- [ ] Immich for self-hosted photo library (replacing Google Photos)
- [ ] Prometheus + Grafana observability stack
- [ ] Ansible roles for remaining services
- [ ] Complete NUT live failover test
- [ ] Azure homelab projects for cloud skills development

---

## Hardware

| Component | Device |
|---|---|
| Hypervisor host | HP Z2 Mini G5 |
| NAS (incoming) | UGREEN DXP4800 Plus |
| UPS | CyberPower CP750SL 750VA Slimline |
| Network | Consumer router (pfSense replacement planned) |
