# Homeserver - Proxmox VE

This section covers how to set up a personal home server using ***[Proxmox VE](https://www.proxmox.com/en/products/proxmox-virtual-environment/overview)***.

Unlike the previous ***[Port Forwarding](../ports)*** setup, where most services ran directly on a single bare-metal Linux installation, this setup uses Proxmox VE as a dedicated hypervisor and separates infrastructure roles between virtual machines and LXC containers.

This approach introduces more complexity, but provides stronger workload isolation, easier backups and recovery, cleaner separation of responsibilities, and more flexibility for experimenting with different operating systems and network layouts.

> [!NOTE]
> This is the **current setup** and is still being actively developed. The architecture, tooling, and service placement may change as the migration progresses.

## Goals & Features

After following this tutorial you will have:

- A dedicated ***Proxmox VE*** hypervisor running directly on bare metal
- Separate virtual machines and LXC containers for infrastructure and application workloads
- A management plane accessible only from trusted LAN/VPN networks
- Public web services exposed through a dedicated reverse proxy instead of directly from the Proxmox host
- Local DNS for private, management, and lab services
- Remote access to the private network using a VPN
- Docker workloads isolated inside a dedicated application VM
- Reproducible infrastructure managed using ***OpenTofu***, ***Ansible***, ***cloud-init***, and ***SOPS + age***
- Monitoring and observability for the hypervisor, guests, containers, and applications
- Independent backups of VM/LXC guests and application data
- An isolated environment for development, CI, security experiments, and other lab workloads

<!-- In the end your server may look like this (diagram made by me in [draw.io](https://draw.io/)): -->

<!-- ![Diagram for homeserver with Proxmox VE](assets/diagram_proxmox.png) -->

## 1. Install Proxmox VE

Start by downloading the latest stable ***[Proxmox VE ISO Installer](https://www.proxmox.com/en/downloads/proxmox-virtual-environment/iso)*** and installing it directly on the server.

The official installation documentation is available in the ***[Proxmox VE Administration Guide](https://pve.proxmox.com/pve-docs/pve-admin-guide.html)***.

For the reference system used in this repository, the host storage layout is based on:

```text
ext4 + LVM + LVM-thin
```

This provides a simple layout for a single-NVMe home server:

```text
Proxmox VE
├── local       # Host filesystem, ISO images, templates and snippets
└── local-lvm   # VM and LXC disks
```

ZFS is an excellent option when its features are required, especially with multiple disks, but it is not necessary for every single-disk homelab.

### 1a. Basic Host Configuration

During installation, configure the Proxmox host with a static management address and keep it inside your trusted LAN.

Example configuration used in this setup:

| Setting | Example |
|---|---|
| **Hostname** | `pve01` |
| **FQDN** | `pve01.mgmt.your-domain.tld` |
| **Management IP** | `192.168.50.10/24` |
| **Gateway** | `192.168.50.1` |
| **DNS** | `1.1.1.1` initially, later replaced with the local DNS server |
| **Network interface** | Wired Ethernet |

The management IP should be outside the normal DHCP pool or reserved so that it never changes.

Use a proper hostname and FQDN rather than a generic name such as `proxmox`.

A simple naming convention is:

```text
pve01.mgmt.your-domain.tld
dns01.home.your-domain.tld
vpn01.home.your-domain.tld
apps01.home.your-domain.tld
```

For the initial installation, using a public resolver such as `1.1.1.1` is sufficient. Once the local DNS server is deployed, Proxmox can use that resolver instead.

Use a strong root password and keep SSH access restricted to trusted networks.

After installation, access the Web UI at:

```text
https://192.168.50.10:8006
```

or, once internal DNS is configured:

```text
https://pve01.mgmt.your-domain.tld:8006
```

> [!WARNING]
> Do **not** expose port `8006` directly to the Internet.
>
> The Proxmox management interface should only be reachable from trusted LAN or VPN networks.

For a single-node homelab, you do not need to create a Proxmox cluster.

### 1b. Package Repositories and Updates

After the initial installation:

- Configure the Proxmox repository appropriate for your installation.
- Disable repositories you do not use.
- Update the system.
- Reboot if required after kernel or other important system updates.

Refer to the official ***[Proxmox VE package repositories documentation](https://pve.proxmox.com/wiki/Package_Repositories)*** before changing repository configuration.

The Proxmox host should remain as minimal as possible. Avoid installing application stacks, Docker, databases, reverse proxies, or other unrelated services directly on the hypervisor.

## 2. Network, DNS and Remote Access

Proxmox VE uses Linux networking and normally connects guests to the physical network through a Linux bridge such as `vmbr0`.

A simple home network may look like this:

```text
Internet
   |
Router
   |
LAN
   |
vmbr0
   |
   +-- Proxmox management
   +-- Core infrastructure
   +-- Application VM
   +-- Home automation
   +-- CI / development
   +-- Lab workloads
```

The physical network interface is attached to the bridge, while VM and LXC interfaces connect to it as virtual network interfaces.

### 2a. Public IP and Port Forwarding

The current setup exposes selected public services using traditional port forwarding.

If your ISP offers a static public IP address, it simplifies the setup. If your public IP is dynamic, use ***[DDNS](https://en.wikipedia.org/wiki/Dynamic_DNS)*** to keep the required DNS records synchronized with your current address.

Your Internet connection must provide a publicly reachable IP address and must not be behind CGNAT if you want to expose services directly using port forwarding.

Proxmox VE itself does not require a public IP address. This requirement comes from the architecture used in this setup to expose selected services directly through port forwarding.

The router should forward only the ports required by services running inside the virtualized environment.

A typical configuration may look like this:

| Service name | External port | Internal port | Target | Protocol | Note |
|:---:|:---:|:---:|:---:|:---:|:---:|
| HTTP | 80 | 80 | REVERSE_PROXY_INTERNAL_IP | TCP | Reverse proxy |
| HTTPS | 443 | 443 | REVERSE_PROXY_INTERNAL_IP | TCP/UDP | Reverse proxy / HTTP3 |
| WireGuard VPN | 51820 | 51820 | VPN_INTERNAL_IP | UDP | VPN |
| Minecraft Java | 25565 | 25565 | GAME_SERVER_INTERNAL_IP | TCP/UDP | Optional |
| Minecraft Bedrock | 19132 | 19132 | GAME_SERVER_INTERNAL_IP | UDP | Optional |

> [!IMPORTANT]
> Port forwarding should target the appropriate VM or LXC guest, **not the Proxmox host**.

Do not forward the Proxmox Web UI, SSH, or other management services directly from the Internet.

### 2b. DNS

Public and private DNS should be treated separately.

Public DNS records are used for services intentionally available from the Internet.

Private DNS is used for internal services, management interfaces, and lab environments that should only be reachable from trusted networks.

A practical naming model is:

```text
service.your-domain.tld
service.home.your-domain.tld
host.mgmt.your-domain.tld
host.lab.your-domain.tld
```

For example:

- `*.your-domain.tld` - public services
- `*.home.your-domain.tld` - private services
- `*.mgmt.your-domain.tld` - management interfaces
- `*.lab.your-domain.tld` - development and laboratory environments

In the current architecture, local DNS is planned around ***[Technitium DNS Server](https://technitium.com/dns/)***.

### 2c. VPN

A VPN provides remote access to services that should not be publicly exposed.

Typical examples include:

- Proxmox VE management
- SSH
- internal dashboards
- monitoring interfaces
- DNS administration
- storage administration
- development services
- Home Assistant administration

***[WireGuard](https://www.wireguard.com/)*** is a good fit for this purpose.

In this setup, the VPN is intended to run inside a dedicated VM or LXC guest rather than directly on the Proxmox host.

## 3. Virtual Machines and LXC Containers

One of the main goals of this setup is to keep the Proxmox host focused exclusively on virtualization.

Applications should run inside dedicated guests.

The target architecture is currently divided into the following roles:

| Role | Guest type | Purpose |
|---|---|---|
| **DNS / Core Infrastructure** | LXC | Local DNS and other lightweight core network services |
| **VPN** | LXC or VM | Remote access to the private network |
| **Applications** | VM | Main Docker host for selfhosted applications |
| **Home Assistant** | VM | Home automation environment |
| **CI / Development** | VM | CI runners, development services, and build workloads |
| **Lab** | VM/LXC | Isolated testing, security experiments, and temporary workloads |

The exact number of guests is not important. The goal is to separate workloads when there is a practical reason to do so without turning a small home server into an unnecessarily complicated enterprise environment.

### Virtual Machines

Use virtual machines when you need:

- stronger isolation,
- a separate kernel,
- full operating system control,
- Docker or other container runtimes,
- appliance-style operating systems such as Home Assistant OS,
- kernel-specific configuration,
- PCI/GPU passthrough.

### LXC Containers

Use LXC containers for lightweight infrastructure services that do not require their own kernel.

They are particularly useful for small always-on services such as DNS, VPN, or other network utilities.

> [!TIP]
> Avoid choosing VM or LXC based only on resource usage. Isolation requirements, maintainability, application support, and recovery procedures are usually more important than saving a small amount of RAM.

## 4. Infrastructure as Code

The Proxmox setup is intended to be reproducible instead of being rebuilt manually from screenshots and notes.

Infrastructure automation for this setup is being developed in a separate repository:

**[PoProstuWitold/homelab-infra](https://github.com/PoProstuWitold/homelab-infra)**

> [!NOTE]
> The infrastructure repository is currently a **work in progress**.

The toolchain is divided by responsibility:

| Tool | Responsibility |
|---|---|
| **OpenTofu** | Provision VM/LXC resources in Proxmox |
| **Ansible** | Configure the Proxmox host and guest operating systems |
| **cloud-init** | Bootstrap newly created virtual machines |
| **SOPS + age** | Store encrypted configuration secrets |
| **Docker Compose** | Deploy application stacks inside the application VM |

The intended deployment flow is:

```text
Proxmox VE installation
        |
        v
Ansible host configuration
        |
        v
OpenTofu VM/LXC provisioning
        |
        v
cloud-init guest bootstrap
        |
        v
Ansible guest configuration
        |
        v
Docker Compose application deployment
```

Not every part of the physical Proxmox host should be managed by OpenTofu.

Foundational settings such as the management IP address and physical bridge configuration should remain deliberately simple because accidentally destroying management connectivity has a much larger blast radius than recreating a guest.

## 5. Services

Most application services are intended to run inside a dedicated Docker VM rather than directly on the Proxmox host.

Core infrastructure services may instead receive their own VM or LXC guest.

### Core Infrastructure

- **[Technitium DNS Server](https://technitium.com/dns/)** - local DNS server for internal zones, recursive resolution, and network-wide DNS management.
- **[WireGuard](https://www.wireguard.com/)** - secure remote access to the private network.
- **[Traefik](https://traefik.io/traefik/)** - reverse proxy for publicly exposed web applications.
- **[Authentik](https://goauthentik.io/)** - identity provider and authentication layer.
- **[CrowdSec](https://www.crowdsec.net/)** - collaborative security engine for detecting and responding to malicious traffic.

### Monitoring and Observability

- **[Grafana](https://grafana.com/)** - dashboards and visualization.
- **[Prometheus](https://prometheus.io/)** - metrics collection.
- **[Loki](https://grafana.com/oss/loki/)** - log aggregation.
- **[Grafana Alloy](https://grafana.com/docs/alloy/latest/)** - telemetry collection.
- **[Node Exporter](https://github.com/prometheus/node_exporter)** - host metrics.
- **[cAdvisor](https://github.com/google/cadvisor)** - container metrics.
- **[smartctl_exporter](https://github.com/prometheus-community/smartctl_exporter)** - disk health metrics.
- **[Uptime Kuma](https://github.com/louislam/uptime-kuma)** - service availability monitoring.

### Files and Storage

- **[FileBrowser Quantum](https://filebrowserquantum.com/)** - browser-based file management.
- **[SFTPGo](https://github.com/drakkan/sftpgo)** - SFTP server with Web UI and optional FTP/S and WebDAV support.

### Development

- **[Gitea](https://about.gitea.com/)** - selfhosted Git service.
- **[Diun](https://crazymax.dev/diun/)** - notifications about Docker image updates.
- **[Renovate](https://docs.renovatebot.com/)** - automated dependency and container image updates.

### Media

- **[Jellyfin](https://jellyfin.org/)** - selfhosted media server.
- **[Seerr](https://github.com/seerr-team/seerr)** - media request and discovery manager.
- **[qBittorrent](https://www.qbittorrent.org/)** - download client.
- **[Prowlarr](https://prowlarr.com/)** - indexer manager.
- **[Radarr](https://radarr.video/)** - movie collection manager.
- **[Sonarr](https://sonarr.tv/)** - TV series collection manager.
- **[Lidarr](https://lidarr.audio/)** - music collection manager.

### Home Automation

- **[Home Assistant](https://www.home-assistant.io/)** - local-first home automation platform.
- **[Mosquitto](https://mosquitto.org/)** - MQTT broker.
- **[Zigbee2MQTT](https://www.zigbee2mqtt.io/)** - Zigbee-to-MQTT bridge.

### Other Services

- **[Linkwarden](https://linkwarden.app/)** - bookmark and webpage archival manager.
- **[Mealie](https://mealie.io/)** - recipe manager.
- **[Omni Tools](https://omnitools.app/)** - browser-based utility collection.
- **[DoggoPaste](https://github.com/PoProstuWitold/doggopaste)** - selfhosted paste and code sharing application.
- **[PufferPanel](https://www.pufferpanel.com/)** - game server management panel.
- **[Homepage](https://gethomepage.dev/)** - dashboard for selfhosted services.

The exact application list is a personal preference. The important architectural difference is that these services are no longer installed directly on the physical server.

## 6. Backups and Disaster Recovery

Virtualization makes guest-level backup and restoration significantly easier, but VM backups should not be the only backup layer.

The backup strategy should cover:

- Proxmox VM and LXC backups
- application configuration
- Docker Compose files
- persistent application data
- database dumps where appropriate
- Infrastructure as Code
- encrypted secrets
- important files stored by applications

Proxmox VE can create VM and LXC backups using its built-in backup system, and ***[Proxmox Backup Server](https://www.proxmox.com/en/products/proxmox-backup-server/overview)*** can be introduced later if a dedicated backup server is available.

Backups should be stored independently from the internal system disk.

At least one additional copy of critical data should exist outside the Proxmox host.

> [!IMPORTANT]
> A backup is only useful if it can be restored.
>
> Periodically test restoration of VM/LXC guests and important application data instead of assuming that a successful backup job guarantees recoverability.
