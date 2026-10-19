# Homeserver

> [!NOTE]
> For more background and context, you can also read my [article about selfhosting](https://witoldzawada.dev/blog/introduction-to-selfhosting).

This repository documents several generations of my home server. The current architecture is based on **Proxmox VE**, AlmaLinux virtual machines, **Podman**, Infrastructure as Code, segmented networking and a dedicated backup target. Older bare-metal Docker setups remain in the repository as stable historical/reference implementations.

The repository currently contains three setups:

| Setup | Status | Description |
| --- | --- | --- |
| **[Proxmox VE](proxmox)** | Current / partially deployed | Architecture: Proxmox VE 9, AlmaLinux VMs, Home Assistant OS, Podman, VLANs and Infrastructure as Code |
| **[Port Forwarding](ports)** | Stable / legacy | Previous bare-metal AlmaLinux + Docker architecture with direct port forwarding and reverse proxying |
| **[Cloudflare Tunnels](tunnels)** | Legacy | Older Cloudflare Tunnel architecture without directly exposing inbound web ports |

> [!IMPORTANT]
> The **[Port Forwarding](ports)** setup represents the final version of the previous bare-metal server. It is retained as a useful reference but is no longer the architecture being developed.

Some details in this guide reflect my own hardware, network and operational preferences. Treat them as a reference design rather than universal requirements.

> [!IMPORTANT]
> The project is designed to avoid recurring infrastructure costs other than electricity and normal domain renewal. Public cloud/VPS dependencies are not required for the current setup.

---

## 0. Things to Consider Before Starting

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
apps01    6 GiB  AlmaLinux - monitoring and application platform
media01   4 GiB  AlmaLinux - media platform (workloads pending)
home01    4 GiB  Home Assistant OS
games01  10 GiB  AlmaLinux - game platform (workloads pending)
```

`lab01` is a provisioned, on-demand 4 GiB AlmaLinux validation VM, not part of the always-on 28 GiB allocation. See the [current deployment status](proxmox/#3-virtual-machines-and-lxc-containers).

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

The legacy guides remain useful as reference implementations, but only the Proxmox section represents the current architecture.

## Infrastructure as Code

Infrastructure automation for the Proxmox setup is maintained separately in **[PoProstuWitold/homelab-infra](https://github.com/PoProstuWitold/homelab-infra)**.

The current infrastructure repository contains the deployed platform foundation, five production VMs and security-focused service configuration:

- **Host and guest hardening:** SSH key-only authentication, restricted root access, SELinux on AlmaLinux, security updates and configuration verifiers.
- **Network security:** VLAN segmentation, Proxmox `INPUT DROP` management firewall and fail-closed guest `firewalld` policies.
- **Identity and secrets:** separate human/automation identities, least-privilege Proxmox API access, Authentik and SOPS + age encrypted machine secrets.
- **Ingress protection:** Traefik on `core01`, DNS-01 TLS certificates, CrowdSec detection, AppSec inspection, live remediation decisions and restricted management routes.
- **Visibility:** VictoriaMetrics, VictoriaLogs, Grafana and Alloy components; additional agents, dashboards and alerts remain in progress.
- **Recovery and repository checks:** dedicated fail-closed VZDump storage, retention, a tested VM restore, pre-commit, Gitleaks and CI validation.

`core01` currently runs DNS, ingress, identity and security services. `apps01` runs the observability backends; `media01` and `games01` still await their application workloads. The full implementation status and evidence links are in [Proxmox security and verification](proxmox/#7-security-controls-and-verification). Future work is tracked in the [infrastructure roadmap](https://github.com/PoProstuWitold/homelab-infra/blob/main/todo.md).

> [!NOTE]
> This repository is an architecture and setup guide. The separate `homelab-infra` repository contains the actual Ansible/OpenTofu implementation and verifiers. Only the explicitly listed services are currently deployed; a documented design is not evidence of a running service.
