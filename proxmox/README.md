# Homeserver - Proxmox VE

This guide covers the Proxmox-based architecture of **[Hestia](../README.md)**, built on **[Proxmox VE](https://www.proxmox.com/en/products/proxmox-virtual-environment/overview)**, AlmaLinux and Podman.

Unlike the earlier **[Port Forwarding](../ports)** setup, applications run inside dedicated virtual machines while the Proxmox host remains focused on virtualization. Five production VMs group services by responsibility: four AlmaLinux guests managed with Ansible and one Home Assistant OS appliance. This provides clear isolation boundaries without creating a separate VM for every application.

## Design Goals

The architecture combines a shared platform foundation with services organized by workload:

- A dedicated **Proxmox VE** hypervisor running directly on bare metal
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
- A dedicated Home Assistant OS VM on the Internal VLAN for home automation
- A dedicated game-server VM sized for an always-on Minecraft server
- Reproducible infrastructure managed with **OpenTofu**, **Ansible**, **cloud-init** and **SOPS + age**
- Fail-closed Proxmox backups stored on a dedicated external SSD
- A provisioned `lab01` VM that starts only on demand when memory headroom permits

## 1. Installing Proxmox VE

Start by downloading the latest stable **[Proxmox VE ISO Installer](https://www.proxmox.com/en/downloads/proxmox-virtual-environment/iso)** and installing it directly on the server.

The official installation documentation is available in the **[Proxmox VE Administration Guide](https://pve.proxmox.com/pve-docs/pve-admin-guide.html)**.

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

### 1.1. Basic Host Configuration

Keep the Proxmox management interface on a trusted, native/untagged LAN rather than a guest/server VLAN.

Reference host configuration:

| Setting | Recommendation |
| --- | --- |
| **Hostname** | `pve01` |
| **Management address** | Static private address outside the DHCP pool |
| **Gateway** | Trusted LAN router |
| **Management network** | Native / untagged trusted LAN |
| **Network interface** | Wired Ethernet |

After installation, access the Web UI at `https://<management-ip>:8006`.

> [!WARNING]
> Do **not** expose port `8006`, Proxmox SSH or console-management ports directly to the Internet.
>
> In the current setup, management access is allowed only from the trusted LAN and explicit WireGuard peers.

The current host uses a dedicated non-root Linux administrator for SSH/sudo, a separate human Proxmox account with TOTP, and a separate least-privilege OpenTofu automation identity.

For a single-node homelab, a Proxmox cluster is not required.

### 1.2. Package Repositories and Updates

After installation:

- Enable the Proxmox no-subscription repository when appropriate for your environment.
- Keep unused enterprise/Ceph repositories disabled.
- Apply system updates.
- Reboot after relevant kernel/platform updates.
- Keep the host minimal.

The current repository automates repository policy, host packages, time synchronization, SSH hardening and RBAC with Ansible.

The host also uses the newer nftables-based `proxmox-firewall` backend. Input defaults to `DROP`; standard management ports and management ping are accepted only from the declared trusted management sources.

Do not install application stacks, databases, reverse proxies or a general-purpose Podman/Docker environment directly on `pve01`.

## 2. Networking, DNS and Remote Access

The current network uses an ASUS RT-BE88U as the router. Proxmox connects through a VLAN-aware `vmbr0`, while host management remains on the native/untagged trusted network.

Network segmentation model:

```text
Native / untagged -> trusted management LAN
VLAN 10           -> Guest / Public
VLAN 20           -> IoT
VLAN 30           -> Home
VLAN 40           -> Internal servers
```

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

### 2.1. Public IP and Port Forwarding

Proxmox VE itself does not require a public IP address. A publicly reachable address is needed only if you intentionally expose services directly from your home connection or want direct inbound WireGuard connectivity.

Current direct publication forwards **TCP and UDP 443** to Traefik on `core01`, never to `pve01`. The current ingress uses DNS-01 ACME, so it does not require an inbound TCP 80 listener.

| Service | External port | Target | Protocol | Status |
| --- | ---: | --- | --- | --- |
| HTTPS / HTTP/3 | 443 | `core01` / Traefik | TCP/UDP | Active |
| WireGuard | router-defined | ASUS router | UDP | Active; private administration |
| Minecraft Java | 25565 | `games01` | TCP | Not published; future workload |

> [!IMPORTANT]
> Port forwarding must target the workload that actually serves the traffic. Never forward the Proxmox Web UI, SSH or console-management ports from the Internet.

If the ISP address is dynamic, DDNS can keep public records synchronized.

### 2.2. DNS

Public and private DNS should remain separate concerns.

The deployed local DNS service is **[Technitium DNS Server](https://technitium.com/dns/)** on `core01`. Trusted clients use it for managed local resolution; other VLANs currently retain their router/default DNS behavior.

A practical naming model is still:

```text
service.your-domain.tld
service.home.your-domain.tld
host.mgmt.your-domain.tld
host.lab.your-domain.tld
```

Private domain names can follow the same structure while remaining independent of public DNS records.

During the earliest Proxmox/OpenTofu bootstrap, do not make provider access depend on a DNS VM that does not exist yet. Bootstrap name resolution and Proxmox CA trust should work independently of Technitium.

### 2.3. VPN

Remote management uses **[WireGuard](https://www.wireguard.com/)** on the ASUS router.

Keeping WireGuard on the router avoids an additional VM or LXC container solely for VPN access.

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

The design keeps `pve01` focused on virtualization; application containers run in guest VMs.

Production v1 uses **five VMs and zero LXC containers**:

| VM | RAM | Role |
| --- | ---: | --- |
| `core01` | 4 GiB | DNS, ingress, identity and security |
| `apps01` | 6 GiB | Monitoring and application platform |
| `media01` | 4 GiB | Media services |
| `home01` | 4 GiB | Home Assistant OS appliance |
| `games01` | 10 GiB | Game servers |

Together, the production VMs reserve 28 GiB out of the host's 32 GiB. The additional `lab01` VM (4 GiB) is used on demand and excluded from the always-on memory budget.

`home01` runs Home Assistant OS rather than an Ansible-managed AlmaLinux installation. Its VM network interface belongs to the Internal VLAN.

`media01` is planned to use Intel Quick Sync through the i5-11400T iGPU for Jellyfin hardware transcoding. Passthrough should be tested before relying on it in production.

`games01` receives 10 GB RAM because the existing Minecraft server uses an 8 GB Java heap. Do not set Minecraft `-Xmx` equal to all VM memory; the JVM, OS, panel and monitoring also need headroom.

### 3.1. Virtual Machines

Use VMs when you need strong isolation, a separate kernel, predictable Ansible management, PCI/iGPU passthrough or a consistent container-host operating system.

The four managed Linux guests use:

```text
Proxmox VM -> AlmaLinux 10 -> cloud-init -> Ansible -> Podman
```

`home01` instead uses `Proxmox VM -> Home Assistant OS -> Supervisor`, with application state managed by HAOS.

### 3.2. LXC Containers

LXC remains a valid Proxmox tool, but it is not currently used in production v1.

Running Podman inside LXC would introduce nested-container/runtime behavior and a second operating model simply to save a relatively small amount of RAM. A future service may use LXC if it has a clear technical advantage, but LXC is not the default optimization knob.

> [!TIP]
> Choose VM vs LXC based on security boundaries, device/kernel requirements and maintainability, not only on idle memory consumption.

## 4. Infrastructure as Code

The `homelab-infra` repository contains the automation for platform provisioning, Linux guest configuration and application deployment. Infrastructure and application responsibilities remain separated, making changes easier to review, test and reproduce.

The toolchain is divided by responsibility:

| Tool | Responsibility |
| --- | --- |
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

Native Proxmox VZDump backups are stored on a dedicated external ext4 filesystem mounted at `/mnt/pve/backup`. The policy backs up production guests by default, with explicit exclusions for reproducible templates and disposable lab VMs.

## 5. Services

Services are grouped by VM rather than giving every container its own guest.

### 5.1. Core Infrastructure

Currently deployed on `core01`:

- **[Technitium DNS Server](https://technitium.com/dns/)** - local zones, recursive resolution and DNS management.
- **[Traefik](https://traefik.io/traefik/)** - reverse proxy for intentionally exposed web applications.
- **[Authentik](https://goauthentik.io/)** - identity provider and SSO/authentication layer.
- **[CrowdSec](https://www.crowdsec.net/)** - detection and response layer for malicious ingress traffic.

WireGuard is not listed here because it runs on the ASUS router, not in a VM.

### 5.2. Monitoring and Observability

The observability backends run on `apps01`:

- **[Grafana](https://grafana.com/)** - dashboards and visualization.
- **[VictoriaMetrics Single](https://victoriametrics.com/)** - metrics storage and querying.
- **[VictoriaLogs Single](https://victoriametrics.com/products/victorialogs/)** - log storage and search.

Current telemetry includes an Alloy agent on `core01` and the central receiver on `apps01`. Metrics and selected operational logs flow to VictoriaMetrics/VictoriaLogs; Grafana queries those backends.

Planned extensions include host agents/exporters, workload dashboards, notifications, Podman metrics and storage health telemetry.

### 5.3. Files and Storage

Planned for `apps01`:

- **[OpenCloud](https://opencloud.eu/)** - selfhosted cloud file platform.

### 5.4. Development

Planned for `apps01`:

- **[Forgejo](https://forgejo.org/)** - selfhosted Git forge.

### 5.5. Media

Planned for `media01`:

- **[Jellyfin](https://jellyfin.org/)** - media server, with planned Intel Quick Sync transcoding.
- **[Seerr](https://github.com/seerr-team/seerr)** - media request/discovery manager.
- **[qBittorrent](https://www.qbittorrent.org/)** - download client.
- **[Prowlarr](https://prowlarr.com/)** - indexer manager.
- **[Radarr](https://radarr.video/)** - movie collection manager.
- **[Sonarr](https://sonarr.tv/)** - TV series collection manager.
- **[Lidarr](https://lidarr.audio/)** - music collection manager.

### 5.6. Home Automation

`home01` runs Home Assistant OS. The home automation stack includes the appliance and planned integrations:

- **[Home Assistant](https://www.home-assistant.io/)** - deployed home automation platform.
- **[Mosquitto](https://mosquitto.org/)** - planned MQTT broker.
- **[Zigbee2MQTT](https://www.zigbee2mqtt.io/)** - planned Zigbee-to-MQTT bridge.

The current Zigbee coordinator is network/Ethernet based, so USB passthrough is not required for that coordinator design.

### 5.7. Other Services

Planned general apps on `apps01`:

- **[Linkwarden](https://linkwarden.app/)** - bookmark and webpage archival manager.
- **[Mealie](https://mealie.io/)** - recipe manager.
- **[Omni Tools](https://omnitools.app/)** - browser-based utility collection.
- **[DoggoPaste](https://github.com/PoProstuWitold/doggopaste)** - selfhosted paste/code sharing application.
- **[Homepage](https://gethomepage.dev/)** - dashboard for selfhosted services.

Planned game services on `games01`:

- **[GameAP](https://gameap.com/)** - game server management panel under evaluation.
- Minecraft - expected to run nearly continuously with an approximately 8 GB maximum Java heap.

GameAP 4 with rootless Podman is the selected proof-of-concept approach; evaluation covers service lifecycle, permissions, persistence and recovery.

## 6. Backups and Disaster Recovery

`pve01` has a dedicated external 2 TB SSD registered as Proxmox storage `backup` and mounted at:

```text
/mnt/pve/backup
```

The mount is validated fail-closed. If the expected filesystem is not present, Proxmox backup storage becomes unavailable instead of silently writing into the host root filesystem.

Current scheduled VZDump policy:

```text
schedule:      daily
mode:          snapshot
compression:   zstd
selection:     production guests by default
exclude:       reproducible templates and on-demand lab VMs
retention:     keep-last=2, keep-daily=7, keep-weekly=4, keep-monthly=3
```

Recovery has been tested with an AlmaLinux VM: the VM was backed up, removed, restored under its original VMID, checked with SSH, QEMU Guest Agent and Ansible, and reconciled with OpenTofu without configuration drift.

Production stateful VMs remain included in the backup-by-default policy. The disposable/on-demand `lab01` is excluded because its state is not treated as persistent.

VM-level backups are not the only layer required for irreplaceable data. Application-aware database dumps and an additional off-host/off-site copy should still be considered for critical data.

> [!IMPORTANT]
> A successful backup job is not proof of recoverability. Periodically perform real restores and verify both VM boot and application-level persistent data.

## 7. Security Controls and Verification

Security is part of the infrastructure design, from isolated workloads and restricted management access to repeatable configuration checks.

| Area | Controls | Verification |
| --- | --- | --- |
| Host and guest hardening | SSH key authentication, restricted root access, SELinux on AlmaLinux and security updates | SSH and privilege checks, SELinux state verification |
| Network security | VLAN segmentation, Proxmox `INPUT DROP`, management allowlists and guest `firewalld` policies | Firewall rule checks and cross-network reachability tests |
| Identity and secrets | Proxmox RBAC, least-privilege automation accounts, Authentik and SOPS + age | Access control checks and encrypted secret handling |
| Ingress | DNS-01 TLS certificates, route exposure policy, CrowdSec remediation and AppSec inspection | TLS, route policy and request-based security checks |
| Observability and recovery | Central metrics and logs, Alloy telemetry, VZDump retention and fail-closed backup storage | Telemetry checks, backup validation and a VM restore exercise |
| Infrastructure automation | Versioned provisioning, linting, policy checks, Gitleaks and pinned container images | Automated repository validation and configuration verifiers |

### 7.1. Validation Workflow

Infrastructure changes follow a repeatable process: validate the configuration, apply it through the appropriate automation tool, and verify the resulting system state. Ingress checks cover route exposure, TLS behavior and CrowdSec/AppSec responses; recovery checks include restoring a VM rather than relying on backup job results alone.
