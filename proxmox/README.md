# Homeserver - Proxmox VE

This section covers how to set up a personal home server using ***[Proxmox VE](https://www.proxmox.com/en/products/proxmox-virtual-environment/overview)***.

Unlike the previous ***[Port Forwarding](../ports)*** setup, where most services ran directly on one bare-metal AlmaLinux installation, the current architecture keeps Proxmox focused on virtualization and runs application workloads inside dedicated AlmaLinux virtual machines using Podman.

The design deliberately avoids unnecessary guest sprawl: production v1 uses five always-on VMs grouped by failure domain and workload type. LXC remains available in Proxmox, but no current v1 workload uses it because a consistent `AlmaLinux -> Ansible -> Podman` model is easier to maintain and recover.

> [!NOTE]
> This is the **current Hestia setup**. The Proxmox host foundation, firewall, backup policy, OpenTofu bootstrap and reference VM workflow are already validated. Production VM provisioning and application migration are the active next phase.

## Goals & Features

After following this tutorial you will have:

- A dedicated ***Proxmox VE*** hypervisor running directly on bare metal
- A minimal Proxmox host with no normal application containers running directly on it
- Five production AlmaLinux VMs grouped into core, apps, media, home automation and game-server roles
- Podman as the standard application container runtime inside production VMs
- A management plane reachable only from the trusted LAN and explicitly allowed WireGuard peers
- Router-hosted WireGuard rather than a dedicated VPN VM
- Public web services exposed through Traefik rather than from the Proxmox host
- Local DNS using Technitium DNS Server
- Authentik-backed identity/SSO and CrowdSec at the ingress/security layer
- Grafana with VictoriaMetrics and VictoriaLogs for observability
- Host telemetry through Grafana Alloy, `prometheus.exporter.unix`, Podman metrics and SMART metrics
- A dedicated media VM with Jellyfin and planned Intel Quick Sync hardware transcoding
- A dedicated smart-home VM with controlled access to both Internal and IoT VLANs
- A dedicated game-server VM sized for an always-on Minecraft server
- Reproducible infrastructure managed with ***OpenTofu***, ***Ansible***, ***cloud-init*** and ***SOPS + age***
- Fail-closed Proxmox backups stored on a dedicated external SSD
- An on-demand lab VM when memory headroom permits

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

The current Hestia repository automates repository policy, host packages, time synchronization, SSH hardening and RBAC with Ansible.

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
          +-- home01 (+ VLAN 20 secondary NIC)
          +-- games01
```

### 2a. Public IP and Port Forwarding

Proxmox VE itself does not require a public IP address. A publicly reachable address is needed only if you intentionally expose services directly from your home connection or want direct inbound WireGuard connectivity.

Public web traffic should be forwarded to **Traefik on `core01`**, never to `pve01`.

Typical direct exposure for the current architecture:

| Service | External port | Target | Protocol | Note |
|---|---:|---|---|---|
| HTTP | 80 | `core01` / Traefik | TCP | Redirect/ACME/application ingress as configured |
| HTTPS | 443 | `core01` / Traefik | TCP/UDP as required | Public web ingress / optional HTTP/3 |
| WireGuard | router-defined | ASUS router | UDP | Remote management terminates on the router |
| Minecraft Java | 25565 | `games01` | TCP | Only if the server is intentionally public |

> [!IMPORTANT]
> Port forwarding must target the workload that actually serves the traffic. Never forward the Proxmox Web UI, SSH or console-management ports from the Internet.

If the ISP address is dynamic, DDNS can keep public records synchronized.

### 2b. DNS

Public and private DNS should remain separate concerns.

The current local DNS service is ***[Technitium DNS Server](https://technitium.com/dns/)*** on `core01`. It is intended to provide local zones, recursive resolution and network-wide DNS management.

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

The current design keeps `pve01` focused exclusively on virtualization and host-level telemetry.

Production v1 intentionally uses **five VMs and zero LXC containers**:

| VMID | Hostname | Internal IP | RAM | Purpose |
|---:|---|---|---:|---|
| `110` | `core01` | `192.168.40.10` | 3 GB | DNS, ingress, identity, edge security |
| `120` | `apps01` | `192.168.40.20` | 6 GB | General apps, cloud, Git, observability backends |
| `130` | `media01` | `192.168.40.30` | 4 GB | Jellyfin and media automation |
| `140` | `home01` | `192.168.40.40` | 2 GB | Home Assistant, MQTT and Zigbee |
| `150` | `games01` | `192.168.40.50` | 10 GB | PufferPanel and Minecraft |

The production allocation is 25 GB out of 32 GB. Remaining RAM is reserved for the hypervisor/QEMU/cache and, when actual memory pressure permits, an on-demand `lab01` around 4 GB.

The current OpenTofu reference VM uses `192.168.40.10`, so it must be removed or renumbered before `core01` is provisioned.

`home01` is the only deliberate dual-homed guest. It will have:

```text
VLAN 40 NIC -> normal Internal address + default route
VLAN 20 NIC -> IoT address, no default route
ip_forward  -> disabled
```

This lets Home Assistant reach IoT devices without making the VM a router between VLANs.

`media01` is planned to use Intel Quick Sync through the i5-11400T iGPU for Jellyfin hardware transcoding. Passthrough should be tested before relying on it in production.

`games01` receives 10 GB RAM because the existing Minecraft server uses an 8 GB Java heap. Do not set Minecraft `-Xmx` equal to all VM memory; the JVM, OS, panel and monitoring also need headroom.

### Virtual Machines

Use VMs when you need strong isolation, a separate kernel, predictable Ansible management, PCI/iGPU passthrough or a consistent container-host operating system.

That is why all current production guests use the same model:

```text
Proxmox VM
  -> AlmaLinux 10
    -> cloud-init
    -> Ansible
    -> Podman
```

### LXC Containers

LXC remains a valid Proxmox tool, but it is not currently used in Hestia production v1.

Running Podman inside LXC would introduce nested-container/runtime behavior and a second operating model simply to save a relatively small amount of RAM. A future service may use LXC if it has a clear technical advantage, but LXC is not the default optimization knob.

> [!TIP]
> Choose VM vs LXC based on security boundaries, device/kernel requirements and maintainability, not only on idle memory consumption.

## 4. Infrastructure as Code

The current architecture is managed in a separate repository:

**[PoProstuWitold/homelab-infra](https://github.com/PoProstuWitold/homelab-infra)**

The host foundation is already implemented and validated. Production workload definitions are the next phase.

The toolchain is divided by responsibility:

| Tool | Responsibility |
|---|---|
| **OpenTofu** | Pinned cloud image, AlmaLinux template and VM lifecycle |
| **Ansible** | Proxmox host desired state and AlmaLinux guest desired state |
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

Current backups use native Proxmox VZDump to a dedicated external ext4 filesystem mounted at `/mnt/pve/backup`. The scheduled policy is backup-by-default with the reproducible template excluded.

## 5. Services

Services are grouped by VM rather than giving every container its own guest.

### Core Infrastructure

Runs on `core01`:

- **[Technitium DNS Server](https://technitium.com/dns/)** - local zones, recursive resolution and DNS management.
- **[Traefik](https://traefik.io/traefik/)** - reverse proxy for intentionally exposed web applications.
- **[Authentik](https://goauthentik.io/)** - identity provider and SSO/authentication layer.
- **[CrowdSec](https://www.crowdsec.net/)** - detection and response layer for malicious ingress traffic.

WireGuard is not listed here because it runs on the ASUS router, not in a VM.

### Monitoring and Observability

Backends run primarily on `apps01`:

- **[Grafana](https://grafana.com/)** - dashboards and visualization.
- **[VictoriaMetrics](https://victoriametrics.com/)** Single - metrics storage/backend.
- **[VictoriaLogs](https://victoriametrics.com/products/victorialogs/)** Single - log storage/search backend.
- **[Uptime Kuma](https://github.com/louislam/uptime-kuma)** - availability monitoring.

Telemetry collection is distributed:

- **[Grafana Alloy](https://grafana.com/docs/alloy/latest/)** on `pve01` and every production VM.
- Alloy `prometheus.exporter.unix` replaces a separate Node Exporter.
- **[prometheus-podman-exporter](https://github.com/containers/prometheus-podman-exporter)** replaces cAdvisor on Podman workload VMs.
- **[smartctl_exporter](https://github.com/prometheus-community/smartctl_exporter)** runs only on `pve01`, where physical disks are visible.

Metrics flow to VictoriaMetrics; journal/container logs flow to VictoriaLogs; Grafana reads both backends.

### Files and Storage

Runs on `apps01`:

- **[OpenCloud](https://opencloud.eu/)** - selfhosted cloud file platform.

The previous SFTPGo/FileBrowser alternatives are not part of the current v1 service list.

### Development

Runs on `apps01`:

- **[Forgejo](https://forgejo.org/)** - selfhosted Git forge.
- **[Renovate](https://docs.renovatebot.com/)** - dependency/container image update automation, intended to run periodically rather than as a permanently active updater.

Gitea and Diun are no longer part of the selected v1 stack.

### Media

Runs on `media01`:

- **[Jellyfin](https://jellyfin.org/)** - media server, with planned Intel Quick Sync transcoding.
- **[Seerr](https://github.com/seerr-team/seerr)** - media request/discovery manager.
- **[qBittorrent](https://www.qbittorrent.org/)** - download client.
- **[Prowlarr](https://prowlarr.com/)** - indexer manager.
- **[Radarr](https://radarr.video/)** - movie collection manager.
- **[Sonarr](https://sonarr.tv/)** - TV series collection manager.
- **[Lidarr](https://lidarr.audio/)** - music collection manager.

### Home Automation

Runs on `home01`:

- **[Home Assistant](https://www.home-assistant.io/)** - local-first home automation platform.
- **[Mosquitto](https://mosquitto.org/)** - MQTT broker.
- **[Zigbee2MQTT](https://www.zigbee2mqtt.io/)** - Zigbee-to-MQTT bridge.

The current Zigbee coordinator is network/Ethernet based, so USB passthrough is not required for that coordinator design.

### Other Services

General apps on `apps01`:

- **[Linkwarden](https://linkwarden.app/)** - bookmark and webpage archival manager.
- **[Mealie](https://mealie.io/)** - recipe manager.
- **[Omni Tools](https://omnitools.app/)** - browser-based utility collection.
- **[DoggoPaste](https://github.com/PoProstuWitold/doggopaste)** - selfhosted paste/code sharing application.
- **[Homepage](https://gethomepage.dev/)** - dashboard for selfhosted services.

Game services on `games01`:

- **[PufferPanel](https://www.pufferpanel.com/)** - game server management panel.
- Minecraft - expected to run nearly continuously with an approximately 8 GB maximum Java heap.

PufferPanel is kept for now, but its Docker-oriented environment is not treated as officially Podman-supported. A Docker-compatible Podman API PoC should verify image lifecycle, start/stop/restart, console, ports, resource limits and restart persistence before production migration.

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
exclude:       reproducible AlmaLinux template (VMID 9000)
retention:     keep-last=2, keep-daily=7, keep-weekly=4, keep-monthly=3
```

The reference AlmaLinux VM has already completed a destructive restore drill: it was backed up, removed outside OpenTofu, restored under the original VMID, verified with SSH/QEMU Guest Agent/Ansible, and reconciled by OpenTofu with no drift.

Production stateful VMs should remain included in the backup-by-default policy. A disposable/on-demand lab VM may be excluded only when its recovery expectations are explicitly documented.

VM-level backups are not the only layer required for irreplaceable data. Application-aware database dumps and an additional off-host/off-site copy should still be considered for critical data.

> [!IMPORTANT]
> A successful backup job is not proof of recoverability. Periodically perform real restores and verify both VM boot and application-level persistent data.
