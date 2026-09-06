# Homeserver

> [!NOTE]
> For more background and context, you can also read my [article about selfhosting](https://witoldzawada.dev/blog/introduction-to-selfhosting).

This repository documents several generations of my home server. The current architecture is based on **Proxmox VE**, AlmaLinux virtual machines, **Podman**, Infrastructure as Code, segmented networking and a dedicated backup target. Older bare-metal Docker setups remain in the repository as stable historical/reference implementations.

The repository currently contains three setups:

| Setup | Status | Description |
| --- | --- | --- |
| **[Proxmox VE](proxmox)** | Current / active development | Current Hestia architecture: Proxmox VE, AlmaLinux VMs, Podman, VLANs, WireGuard on the router and Infrastructure as Code |
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

The current **[Proxmox VE](proxmox)** setup runs Proxmox directly on bare metal. Application workloads run in AlmaLinux VMs and are containerized with Podman. The Proxmox host itself stays minimal and does not run normal application containers.

The current production v1 design uses five always-on VMs:

```text
core01    3 GB
apps01    6 GB
media01   4 GB
home01    2 GB
games01  10 GB
```

An on-demand `lab01` is planned only when host memory headroom allows it.

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

The legacy guides remain useful as reference implementations, but only the Proxmox section represents the current Hestia architecture.

## Infrastructure as Code

Infrastructure automation for the Proxmox setup is maintained separately in **[PoProstuWitold/homelab-infra](https://github.com/PoProstuWitold/homelab-infra)**.

The current infrastructure repository already covers the validated host foundation:

- Proxmox host baseline and SSH hardening with Ansible
- nftables-based `proxmox-firewall` management policy
- least-privilege OpenTofu identity and API token metadata
- SOPS + hybrid post-quantum age secret handling
- pinned AlmaLinux 10 cloud image and reusable VM template
- reference VM provisioning and Ansible guest baseline
- VLAN 40 guest networking
- fail-closed external backup storage
- scheduled VZDump retention policy
- destructive VM backup/restore validation
- pre-commit and Gitleaks repository safeguards

The production five-VM topology and service placement are now defined. Production VM provisioning and service migration are the next implementation phase.
