# Homeserver - Proxmox VE

This section covers how to set up a personal home server using ***[Proxmox VE](https://www.proxmox.com/en/products/proxmox-virtual-environment/overview)***.

Unlike the previous ***[Port Forwarding](../ports)*** setup, where most services ran directly on one bare-metal AlmaLinux installation, the current architecture keeps Proxmox focused on virtualization and runs managed Linux workloads inside AlmaLinux virtual machines with Podman; Home Assistant runs in its own HAOS appliance.

The design deliberately avoids unnecessary guest sprawl: production v1 uses five VMs grouped by workload type. Four run AlmaLinux with Ansible and Podman; `home01` is a dedicated Home Assistant OS appliance. LXC remains available, but is not used for the current production topology.

> [!NOTE]
> This is the **current setup**. All five production VMs are provisioned, with validated host/guest baselines, network policies, backup infrastructure, core ingress/security services and observability backends. Media/game applications and some other services are still pending. The authoritative deployment state and remaining work live in [homelab-infra architecture](https://github.com/PoProstuWitold/homelab-infra/blob/main/docs/architecture.md) and [roadmap](https://github.com/PoProstuWitold/homelab-infra/blob/main/todo.md).

## Goals & Features

The architecture includes a mix of deployed infrastructure and planned workloads:

- A dedicated ***Proxmox VE*** hypervisor running directly on bare metal
- A minimal Proxmox host with no normal application containers running directly on it
- Five production VMs: four managed AlmaLinux guests and one Home Assistant OS appliance
- Podman as the standard application container runtime inside the four managed AlmaLinux production VMs
- A management plane reachable only from the trusted LAN and explicitly allowed WireGuard peers
- Router-hosted WireGuard rather than a dedicated VPN VM
- Public web services exposed through Traefik rather than from the Proxmox host
- Local DNS using Technitium DNS Server
- Authentik-backed identity/SSO and CrowdSec at the ingress/security layer
- Grafana with VictoriaMetrics and VictoriaLogs for observability
- Initial telemetry using Grafana Alloy; additional host/Podman/SMART metrics are planned
- A dedicated media VM with Jellyfin and planned Intel Quick Sync hardware transcoding
- A dedicated Home Assistant OS VM on the Internal VLAN; IoT integration is a future step
- A dedicated game-server VM sized for an always-on Minecraft server
- Reproducible infrastructure managed with ***OpenTofu***, ***Ansible***, ***cloud-init*** and ***SOPS + age***
- Fail-closed Proxmox backups stored on a dedicated external SSD
- A provisioned `lab01` VM that starts only on demand when memory headroom permits

<!-- In the end your server may look like this (diagram made by me in [draw.io](https://draw.io/)): -->

<!-- ![Diagram for homeserver with Proxmox VE](assets/diagram_proxmox.png) -->

## 1. Install Proxmox VE

Start by downloading the latest stable ***[Proxmox VE ISO Installer](https://www.proxmox.com/en/downloads/proxmox-virtual-environment/iso)*** and installing it directly on the server.

The official installation documentation is available in the ***[Proxmox VE Administration Guide](https://pve.proxmox.com/pve-docs/pve-admin-guide.html)***.

The current reference host is a Lenovo ThinkCentre M70q Gen 2 with an Intel Core i5-11400T, 32 GB RAM and a 2 TB NVMe SSD. A separate 2 TB Crucial BX500 SSD is used only for Proxmox VZDump backups.

The primary host storage uses the standard single-disk Proxmox layout:

```text
ext4 + LVM + LVM-thin
```

```text
Proxmox VE
├── local       # ISO/import/template files, not VM backups
├── local-lvm   # VM disks
└── backup      # dedicated external ext4 filesystem for VZDump
```

ZFS is valuable when its checksumming, snapshots, mirrors or RAID features are required, but it is not mandatory for a single-NVMe mini-PC homelab.

### 1a. Basic Host Configuration

Keep the Proxmox management interface on a trusted, native/untagged LAN rather than a guest/server VLAN.

Current reference values:

| Setting | Current setup |
|---|---|
| **Hostname** | `pve01` |
| **Management IP** | `192.168.50.10/24` |
| **Gateway/router** | `192.168.50.1` |
| **Management network** | Native / untagged trusted LAN |
| **Network interface** | Wired Ethernet |

The normal DHCP pool starts above the static infrastructure range, so `pve01` keeps a stable address.

After installation, access the Web UI at:

```text
https://192.168.50.10:8006
```

> [!WARNING]
> Do **not** expose port `8006`, Proxmox SSH or console-management ports directly to the Internet.
>
> In the current setup, management access is allowed only from the trusted LAN and explicit WireGuard peers.

The current host uses a dedicated non-root Linux administrator for SSH/sudo, a separate human Proxmox account with TOTP, and a separate least-privilege OpenTofu automation identity.

For a single-node homelab, a Proxmox cluster is not required.

### 1b. Package Repositories and Updates

After installation:

- Enable the Proxmox no-subscription repository when appropriate for your environment.
- Keep unused enterprise/Ceph repositories disabled.
- Apply system updates.
- Reboot after relevant kernel/platform updates.
- Keep the host minimal.

The current repository automates repository policy, host packages, time synchronization, SSH hardening and RBAC with Ansible.

The host also uses the newer nftables-based `proxmox-firewall` backend. Input defaults to `DROP`; standard management ports and management ping are accepted only from the declared trusted management sources.

Do not install application stacks, databases, reverse proxies or a general-purpose Podman/Docker environment directly on `pve01`.

## 2. Network, DNS and Remote Access

The current network uses an ASUS RT-BE88U as the router. Proxmox connects through a VLAN-aware `vmbr0`, while host management remains on the native/untagged trusted network.

Current network model:

```text
Native / untagged -> trusted management LAN (192.168.50.0/24)
VLAN 10           -> Guest / Public
VLAN 20           -> IoT
VLAN 30           -> Home
VLAN 40           -> Internal servers
```

There is intentionally no VLAN 50.

The Proxmox uplink carries the native network plus tagged VLANs 10, 20, 30 and 40. The VLAN-aware bridge configuration is intentionally maintained as manual host/network state because breaking that path can remove management access to the hypervisor.

Production topology:

```text
Internet
   |
ASUS RT-BE88U
   |-- WireGuard remote management
   |-- native trusted management
   |-- VLAN 20 IoT
   |-- VLAN 30 Home
   |-- VLAN 40 Internal
          |
        vmbr0
          |
          +-- core01
          +-- apps01
          +-- media01
          +-- home01 (HAOS, VLAN 40 only)
          +-- games01
```

### 2a. Public IP and Port Forwarding

Proxmox VE itself does not require a public IP address. A publicly reachable address is needed only if you intentionally expose services directly from your home connection or want direct inbound WireGuard connectivity.

Current direct publication forwards **TCP and UDP 443** to Traefik on `core01`, never to `pve01`. The current ingress uses DNS-01 ACME, so it does not require an inbound TCP 80 listener.

| Service | External port | Target | Protocol | Status |
|---|---:|---|---|---|
| HTTPS / HTTP/3 | 443 | `core01` / Traefik | TCP/UDP | Active |
| WireGuard | router-defined | ASUS router | UDP | Active; private administration |
| Minecraft Java | 25565 | `games01` | TCP | Not published; future workload |

The planned OVH `edge01` / WireGuard / local `transit01` transport has **not** replaced the current router forwarding.

> [!IMPORTANT]
> Port forwarding must target the workload that actually serves the traffic. Never forward the Proxmox Web UI, SSH or console-management ports from the Internet.

If the ISP address is dynamic, DDNS can keep public records synchronized.

### 2b. DNS

Public and private DNS should remain separate concerns.

The deployed local DNS service is ***[Technitium DNS Server](https://technitium.com/dns/)*** on `core01`. Trusted clients use it for managed local resolution; other VLANs currently retain their router/default DNS behavior.

A practical naming model is still:

```text
service.your-domain.tld
service.home.your-domain.tld
host.mgmt.your-domain.tld
host.lab.your-domain.tld
```

The exact private domain naming is environment-specific and should not be hardcoded into a public guide unless necessary.

During the earliest Proxmox/OpenTofu bootstrap, do not make provider access depend on a DNS VM that does not exist yet. Bootstrap name resolution and Proxmox CA trust should work independently of Technitium.

### 2c. VPN

Remote management uses ***[WireGuard](https://www.wireguard.com/)*** on the ASUS router.

This is an important change from the older plan: there is **no dedicated VPN VM/LXC** in production v1.

WireGuard is used for access to services that should remain private, such as:

- Proxmox VE Web UI
- SSH
- DNS administration
- Grafana and internal observability interfaces
- OpenCloud/Forgejo administration when kept private
- Home Assistant administration
- lab workloads

`pve01` receives the WireGuard peer source address directly. Its firewall therefore allowlists explicit peer `/32` addresses rather than trusting the entire VPN subnet.

## 3. Virtual Machines and LXC Containers

The current design keeps `pve01` focused on virtualization. Dedicated host telemetry is planned; application containers do not run there.

Production v1 intentionally uses **five VMs and zero LXC containers**:

| VMID | Hostname | Internal IP | RAM | Purpose |
|---:|---|---|---:|---|
| `110` | `core01` | `192.168.40.10` | 4 GiB | Deployed DNS, ingress, identity, security |
| `120` | `apps01` | `192.168.40.20` | 6 GiB | Deployed monitoring and log backends |
| `130` | `media01` | `192.168.40.30` | 4 GiB | Platform ready; media workloads pending |
| `140` | `home01` | `192.168.40.40` | 4 GiB | Home Assistant OS appliance |
| `150` | `games01` | `192.168.40.50` | 10 GiB | Platform ready; game workloads pending |

The five provisioned production VMs allocate 28 GiB out of 32 GiB. `lab01` (VMID `190`, 4 GiB, `192.168.40.90`) is also provisioned but powered off by default and excluded from the always-on budget. Check real memory headroom before starting it.

`home01` is an HAOS appliance, **not** an Ansible-managed AlmaLinux guest. Its current OpenTofu configuration has one VLAN 40 interface. Home Assistant onboarding and IoT integrations are still pending; do not assume a VLAN 20 NIC or completed IoT access.

`media01` is planned to use Intel Quick Sync through the i5-11400T iGPU for Jellyfin hardware transcoding. Passthrough should be tested before relying on it in production.

`games01` receives 10 GB RAM because the existing Minecraft server uses an 8 GB Java heap. Do not set Minecraft `-Xmx` equal to all VM memory; the JVM, OS, panel and monitoring also need headroom.

### Virtual Machines

Use VMs when you need strong isolation, a separate kernel, predictable Ansible management, PCI/iGPU passthrough or a consistent container-host operating system.

The four managed Linux guests use:

```text
Proxmox VM -> AlmaLinux 10 -> cloud-init -> Ansible -> Podman
```

`home01` instead uses `Proxmox VM -> Home Assistant OS -> Supervisor`, with application state managed by HAOS.

### LXC Containers

LXC remains a valid Proxmox tool, but it is not currently used in production v1.

Running Podman inside LXC would introduce nested-container/runtime behavior and a second operating model simply to save a relatively small amount of RAM. A future service may use LXC if it has a clear technical advantage, but LXC is not the default optimization knob.

> [!TIP]
> Choose VM vs LXC based on security boundaries, device/kernel requirements and maintainability, not only on idle memory consumption.

## 4. Infrastructure as Code

The current architecture is managed in a separate repository:

**[PoProstuWitold/homelab-infra](https://github.com/PoProstuWitold/homelab-infra)**

The host foundation, production VM definitions, Linux guest baselines and core/observability services are implemented. The remaining application workloads and edge-network migration are future work.

The toolchain is divided by responsibility:

| Tool | Responsibility |
|---|---|
| **OpenTofu** | Pinned cloud image, AlmaLinux template and VM lifecycle |
| **Ansible** | Proxmox host, AlmaLinux guests and deployed service configuration |
| **cloud-init** | First-boot user, SSH and network bootstrap |
| **SOPS + age** | Encrypted secrets and selected OPSEC-sensitive values |
| **Podman** | Application containers inside production VMs |
| **pre-commit + Gitleaks** | Repository secret and hygiene safeguards |

The deployment flow is:

```text
manual Proxmox installation
        |
        v
Ansible host baseline + firewall + backup policy
        |
        v
OpenTofu image/template/VM provisioning
        |
        v
cloud-init first boot
        |
        v
Ansible guest baseline
        |
        v
Podman application deployment
        |
        v
VictoriaMetrics / VictoriaLogs / Grafana verification
```

The router VLAN policy and the physical Proxmox bridge remain intentionally outside normal OpenTofu/Ansible convergence. Per-VM VLAN tags belong in OpenTofu.

Current backups use native Proxmox VZDump to a dedicated external ext4 filesystem mounted at `/mnt/pve/backup`. The scheduled policy is backup-by-default, excluding the reproducible template and disposable `lab01`. Specific nonessential data disks are also excluded as described in the [infrastructure architecture](https://github.com/PoProstuWitold/homelab-infra/blob/main/docs/architecture.md).

## 5. Services

Services are grouped by VM rather than giving every container its own guest.

### Core Infrastructure

Currently deployed on `core01`:

- **[Technitium DNS Server](https://technitium.com/dns/)** - local zones, recursive resolution and DNS management.
- **[Traefik](https://traefik.io/traefik/)** - reverse proxy for intentionally exposed web applications.
- **[Authentik](https://goauthentik.io/)** - identity provider and SSO/authentication layer.
- **[CrowdSec](https://www.crowdsec.net/)** - detection and response layer for malicious ingress traffic.

WireGuard is not listed here because it runs on the ASUS router, not in a VM.

### Monitoring and Observability

The observability backends run on `apps01`:

- **[Grafana](https://grafana.com/)** - dashboards and visualization.
- **[VictoriaMetrics](https://victoriametrics.com/)** Single - metrics storage/backend.
- **[VictoriaLogs](https://victoriametrics.com/products/victorialogs/)** Single - log storage/search backend.

Current telemetry includes an Alloy agent on `core01` and the central receiver on `apps01`. Metrics and selected operational logs flow to VictoriaMetrics/VictoriaLogs; Grafana queries those backends.

Further host agents/exporters (including `pve01` and other workload guests), focused dashboards and notifications are **planned**, not yet deployed. The selected future approach includes Alloy host metrics, Podman exporters and storage health telemetry.

### Files and Storage

Planned for `apps01` (not yet deployed):

- **[OpenCloud](https://opencloud.eu/)** - selfhosted cloud file platform.

The previous SFTPGo/FileBrowser alternatives are not part of the current v1 service list.

### Development

Planned for `apps01` (not yet deployed):

- **[Forgejo](https://forgejo.org/)** - selfhosted Git forge.

### Media

Planned for `media01` (not yet deployed):

- **[Jellyfin](https://jellyfin.org/)** - media server, with planned Intel Quick Sync transcoding.
- **[Seerr](https://github.com/seerr-team/seerr)** - media request/discovery manager.
- **[qBittorrent](https://www.qbittorrent.org/)** - download client.
- **[Prowlarr](https://prowlarr.com/)** - indexer manager.
- **[Radarr](https://radarr.video/)** - movie collection manager.
- **[Sonarr](https://sonarr.tv/)** - TV series collection manager.
- **[Lidarr](https://lidarr.audio/)** - music collection manager.

### Home Automation

`home01` runs Home Assistant OS and its appliance runtime, including Home Assistant. Onboarding and other integrations are still in progress:

- **[Home Assistant](https://www.home-assistant.io/)** - appliance runtime deployed; setup/integration pending.
- **[Mosquitto](https://mosquitto.org/)** - MQTT broker.
- **[Zigbee2MQTT](https://www.zigbee2mqtt.io/)** - Zigbee-to-MQTT bridge.

The current Zigbee coordinator is network/Ethernet based, so USB passthrough is not required for that coordinator design.

### Other Services

Planned general apps on `apps01`:

- **[Linkwarden](https://linkwarden.app/)** - bookmark and webpage archival manager.
- **[Mealie](https://mealie.io/)** - recipe manager.
- **[Omni Tools](https://omnitools.app/)** - browser-based utility collection.
- **[DoggoPaste](https://github.com/PoProstuWitold/doggopaste)** - selfhosted paste/code sharing application.
- **[Homepage](https://gethomepage.dev/)** - dashboard for selfhosted services.

Planned game services on `games01`:

- **[GameAP](https://gameap.com/)** - proposed game server panel; PoC pending.
- Minecraft - expected to run nearly continuously with an approximately 8 GB maximum Java heap.

GameAP 4 with rootless Podman is the current PoC plan. Its integration, lifecycle, permissions, persistence and backup/restore behavior must be validated before production exposure.

## 6. Backups and Disaster Recovery

The current backup design is already more concrete than the previous version of this guide.

`pve01` has a dedicated external 2 TB SSD registered as Proxmox storage `backup` and mounted at:

```text
/mnt/pve/backup
```

The mount is validated fail-closed. If the expected filesystem is not present, Proxmox backup storage becomes unavailable instead of silently writing into the host root filesystem.

Current scheduled VZDump policy:

```text
schedule:      daily at 03:00
mode:          snapshot
compression:   zstd
selection:     all guests by default
exclude:       reproducible template (VMID 9000) and on-demand lab01 (VMID 190)
retention:     keep-last=2, keep-daily=7, keep-weekly=4, keep-monthly=3
```

The reference AlmaLinux VM has already completed a destructive restore drill: it was backed up, removed outside OpenTofu, restored under the original VMID, verified with SSH/QEMU Guest Agent/Ansible, and reconciled by OpenTofu with no drift.

Production stateful VMs remain included in the backup-by-default policy. The disposable/on-demand `lab01` is excluded because its state is not treated as persistent.

VM-level backups are not the only layer required for irreplaceable data. Application-aware database dumps and an additional off-host/off-site copy should still be considered for critical data.

> [!IMPORTANT]
> A successful backup job is not proof of recoverability. Periodically perform real restores and verify both VM boot and application-level persistent data.


## 7. Security Controls and Verification

The separate [`homelab-infra`](https://github.com/PoProstuWitold/homelab-infra) repository contains the actual security configuration, automation and checks. The table below links to implementation evidence rather than treating a list of installed applications as proof of security.

| Control | Implemented approach | Configuration / verification |
| --- | --- | --- |
| Host and guest hardening | SSH key authentication, root-login restrictions, SELinux for managed AlmaLinux guests, security updates | [`proxmox_host`](https://github.com/PoProstuWitold/homelab-infra/tree/main/ansible/roles/platform/proxmox_host), [`guest_baseline`](https://github.com/PoProstuWitold/homelab-infra/tree/main/ansible/roles/platform/guest_baseline) |
| Firewalls and segmentation | VLAN-aware bridge, Proxmox `INPUT DROP`, restricted management sources, guest `firewalld` policies and reachability checks | [`proxmox_firewall`](https://github.com/PoProstuWitold/homelab-infra/tree/main/ansible/roles/platform/proxmox_firewall), [`guest_firewall`](https://github.com/PoProstuWitold/homelab-infra/tree/main/ansible/roles/platform/guest_firewall) |
| Identity and secrets | Proxmox RBAC and limited OpenTofu API identity, Authentik identity service, SOPS + age encrypted deployment values | [`Proxmox RBAC`](https://github.com/PoProstuWitold/homelab-infra/blob/main/ansible/roles/platform/proxmox_host/tasks/rbac.yml), [`authentik`](https://github.com/PoProstuWitold/homelab-infra/blob/main/docs/services/authentik.md) |
| HTTP ingress protection | TLS with DNS-01, route-level exposure policy, CrowdSec LAPI remediation and AppSec inspection | [`traefik`](https://github.com/PoProstuWitold/homelab-infra/blob/main/docs/services/traefik.md), [`crowdsec`](https://github.com/PoProstuWitold/homelab-infra/blob/main/docs/services/crowdsec.md), [`live verification`](https://github.com/PoProstuWitold/homelab-infra/blob/main/ansible/roles/services/crowdsec_traefik_bouncer/tasks/verify-live.yml) |
| Monitoring and recovery | Central metrics/log storage, selected Alloy collection, VZDump retention with fail-closed external mount and a VM restore drill | [`architecture`](https://github.com/PoProstuWitold/homelab-infra/blob/main/docs/architecture.md), [`observability`](https://github.com/PoProstuWitold/homelab-infra/blob/main/docs/services/observability.md) |
| Automation integrity | Encrypted source secrets, lint/format/policy checks, Gitleaks and pinned container images | [`validation workflow`](https://github.com/PoProstuWitold/homelab-infra/blob/main/.github/workflows/validate.yml), [`security policy`](https://github.com/PoProstuWitold/homelab-infra/blob/main/SECURITY.md) |

### Validation model

Infrastructure changes pass through explicit ownership boundaries: review and validate the change, apply it with the appropriate tool, then run independent Ansible verifiers where available. Ingress validation includes checks of route exposure, TLS and request-driven CrowdSec/AppSec behavior. A green configuration test **does not** replace an audit or prove that every possible attack is blocked.

### Current limitations

- Additional monitoring agents, alerting, off-host application-aware backups and the OVH edge migration remain future work.
- LUKS2/VeraCrypt on personal operator devices are outside this server Infrastructure-as-Code repository and are not claimed here as server-side controls.

For the authoritative implementation status, consult [`docs/architecture.md`](https://github.com/PoProstuWitold/homelab-infra/blob/main/docs/architecture.md) and [`todo.md`](https://github.com/PoProstuWitold/homelab-infra/blob/main/todo.md) in the infrastructure repository.
