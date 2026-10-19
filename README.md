# Homeserver

**Hestia** is the name I gave my home server. In Greek mythology, Hestia is the goddess of the hearth and home - a fitting name for the infrastructure that brings my selfhosted services and data together.

This repository documents the project's evolution, from Docker-based servers to a Proxmox VE architecture with AlmaLinux virtual machines, Podman, network segmentation, Infrastructure as Code and dedicated backups. The focus is on practical selfhosting, maintainability and control over the infrastructure.

> [!NOTE]
> For more background, see my [introduction to selfhosting](https://witoldzawada.dev/blog/introduction-to-selfhosting).

## Architectures

| Architecture | Role | Description |
| --- | --- | --- |
| **[Proxmox VE](proxmox)** | Primary | Proxmox VE 9, AlmaLinux VMs, Home Assistant OS, Podman, VLANs and Infrastructure as Code |
| **[Port Forwarding](ports)** | Previous generation | Bare-metal AlmaLinux and Docker with direct port forwarding and a reverse proxy |
| **[Cloudflare Tunnels](tunnels)** | Archive | Earlier Docker-based setup using Cloudflare Tunnels for web access |

> [!IMPORTANT]
> The **[Port Forwarding](ports)** guide documents the final version of the previous bare-metal server. It remains available as a reference; the **[Proxmox VE](proxmox)** architecture is the primary architecture for Hestia.
>
> The project is designed to run without recurring public-cloud or VPS expenses. Beyond the hardware itself, ongoing costs are mainly electricity and normal domain renewal.

The hardware choices and examples below reflect a particular setup, not universal requirements.

---

## Before You Start

Before installing a hypervisor or applications, plan the hardware, physical placement, network segmentation, storage, backup target and exposure model.

### Power Consumption

A home server normally runs 24/7, so idle efficiency matters more than peak benchmark performance.

The current reference host is a **[Lenovo ThinkCentre M70q Gen 2](https://pcsupport.lenovo.com/us/en/products/desktops-and-all-in-ones/thinkcentre-m-series-desktops/thinkcentre-m70q-gen-2/documentation/?linkTrack=footer%3ASupport_Manuals)**:

| Component | Current specification |
| --- | --- |
| **CPU** | Intel Core i5-11400T |
| **RAM** | 32 GB DDR4 |
| **Primary storage** | 2 TB NVMe SSD |
| **Backup storage** | 2 TB Crucial BX500 SSD |
| **Network** | Wired Ethernet to an ASUS RT-BE88U router |

This is enough for several always-on application VMs, an 8 GB-heap Minecraft server, monitoring and occasional lab VMs while remaining much more power-efficient than typical rack hardware.

Without a wall power meter, software-side readings are only estimates. Actual system draw depends on storage, peripherals, transcode activity, game-server load and CPU power states.

### Hardware

A practical baseline for a modern home server:

| Component | Recommendation |
| --- | --- |
| **CPU** | 4+ modern x86-64 cores; more if running game servers, transcoding or build workloads |
| **GPU** | Optional; integrated Intel graphics are useful for Jellyfin Quick Sync without a dedicated GPU |
| **RAM** | 16 GB is comfortable for a small server; 32 GB is preferable when using multiple VMs or game servers |
| **Primary storage** | SSD/NVMe strongly recommended |
| **Backup storage** | Separate physical device, ideally with an additional off-host/off-site copy for critical data |

More RAM is useful only when the workload justifies it. The current 32 GB host is intentionally budgeted rather than blindly overprovisioned.

### Noise

Mini PCs and business-class SFF systems are a good fit for a room or home office because they are generally quieter and more efficient than repurposed enterprise rack servers.

Use wired Ethernet for the server whenever possible.

### Physical Size

Leave enough ventilation and physical access for maintenance. External backup disks and future storage expansion also need space and reliable cabling.

### Recommended Steps

Before installing anything:

- Update BIOS/UEFI and relevant firmware.
- Enable CPU virtualization.
- Enable IOMMU/VT-d when PCI/iGPU passthrough may be required.
- Decide where the server will physically live.
- Plan primary storage and an independent backup target.
- Plan VLANs, trusted management access and local DNS.
- Decide which services are public and which remain LAN/VPN-only.
- Prefer wired Ethernet.
- Decide whether the current **[Proxmox VE](proxmox)** architecture or one of the legacy approaches better matches your needs.

### Requirements

#### Common Requirements

- A machine capable of running the selected setup.
- Basic Linux command-line knowledge.
- Wired network access where possible.
- A domain if you want friendly public/private service names.
- A backup plan before storing irreplaceable data.

#### Docker-based Setups

The legacy **[Cloudflare Tunnels](tunnels)** and **[Port Forwarding](ports)** guides use Docker.

#### Proxmox VE

The current **[Proxmox VE](proxmox)** setup runs Proxmox directly on bare metal. Managed Linux workloads run in AlmaLinux VMs with Podman; Home Assistant runs as a separate Home Assistant OS appliance. The Proxmox host itself stays minimal and does not run normal application containers.

The deployed production v1 topology defines five always-on VMs:

```text
core01    4 GiB  AlmaLinux - DNS, ingress, identity and security
apps01    6 GiB  AlmaLinux - monitoring and applications
media01   4 GiB  AlmaLinux - media services
home01    4 GiB  Home Assistant OS
games01  10 GiB  AlmaLinux - game servers
```

`lab01` is an on-demand 4 GiB AlmaLinux VM for validation and experiments, outside the 28 GiB always-on memory allocation. See [Virtual Machines and LXC Containers](proxmox/#3-virtual-machines-and-lxc-containers).

#### Remote Access Requirements

| Setup | Public IP required | Inbound ports required |
| --- | ---: | ---: |
| **Cloudflare Tunnels** | No | No |
| **Port Forwarding** | Yes | Yes |
| **Proxmox VE** | Only for direct public exposure / router VPN reachability | Only for intentionally exposed services |

The Proxmox management interface itself does **not** need a public IP and must not be port-forwarded. In the current setup, remote management is provided by WireGuard running on the ASUS router.

The current Proxmox architecture can expose selected public applications through Traefik using router port forwarding, while management remains private.

---

## Repository Structure

```text
.
├── tunnels/     # Legacy Cloudflare Tunnel setup
├── ports/       # Stable legacy bare-metal Docker setup
└── proxmox/     # Current Proxmox + VM + Podman architecture
```

The Proxmox section describes the primary architecture. The other guides preserve earlier design approaches and their configuration examples.

## Infrastructure as Code

Project Hestia separates architecture documentation from infrastructure automation. The Proxmox environment is provisioned and managed with **OpenTofu**, **Ansible**, **cloud-init** and **SOPS + age**, with applications running in Podman containers inside dedicated VMs.

The accompanying `homelab-infra` repository contains the implementation, configuration and validation workflows. Responsibilities are divided into:

- **Platform:** Proxmox host configuration, VM provisioning, guest baselines and updates.
- **Networking and access:** VLAN segmentation, host and guest firewalls, SSH hardening and least-privilege identities.
- **Ingress and identity:** Traefik, TLS certificates, Authentik and CrowdSec.
- **Observability:** Grafana, VictoriaMetrics, VictoriaLogs and Alloy.
- **Recovery and quality:** VZDump backups, restore testing, automated checks and secret scanning.

For the technical architecture and operational details, continue to the **[Proxmox VE guide](proxmox)**.
