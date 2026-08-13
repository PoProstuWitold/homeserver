# Homeserver

> [!NOTE]
> For more background and context, you can also read my [article about selfhosting](https://witoldzawada.dev/blog/introduction-to-selfhosting).

This guide walks through building and maintaining a home server using **Linux**, **Docker**, virtualization, and different approaches to remote access and service exposure.

The repository currently contains three setups:

| Setup | Status | Description |
| --- | --- | --- |
| **[Proxmox VE](proxmox)** | Current / in progress | The current setup, built around Proxmox VE, virtual machines, LXC containers, and Infrastructure as Code |
| **[Port Forwarding](ports)** | Stable / legacy | A complete bare-metal setup based on AlmaLinux, Docker, direct port forwarding, reverse proxying, and VPN access |
| **[Cloudflare Tunnels](tunnels)** | Legacy | An older setup using Cloudflare Tunnels to expose services without directly opening ports on the home network |

> [!IMPORTANT]
> The **[Port Forwarding](ports)** setup remains a complete and practical implementation of a traditional bare-metal home server. It represents the final version of my previous architecture and is kept as a stable reference, but it is no longer actively developed or updated.

Some parts of this guide reflect my personal preferences, such as Linux distributions, hardware, directory structure, networking, virtualization, and services. Your setup may differ depending on your hardware, network, and use case.

> [!IMPORTANT]
> This guide is intended to stay free of recurring costs, except for power consumption, domain renewal, and one-time hardware purchases such as a router or server device.
>
> For this reason, VPS-based setups and subscription-based services are not covered, or at least are not planned for now.

---

## 0. Things to Consider Before Starting

Before setting up a home server, it is worth planning the hardware, power usage, noise level, physical placement, network access, storage, and basic requirements.

### Power Consumption

A home server usually runs 24/7, so power efficiency matters from the beginning.

You can use an older PC or a compact business-class mini PC. Commonly used models include **Lenovo ThinkCentre**, **Dell OptiPlex**, and **HP EliteDesk**.

The reference system used throughout this guide is a **[Lenovo ThinkCentre M70q Gen 2](https://pcsupport.lenovo.com/us/en/products/desktops-and-all-in-ones/thinkcentre-m-series-desktops/thinkcentre-m70q-gen-2/documentation/?linkTrack=footer%3ASupport_Manuals)** equipped with:

| Component | Specification |
| --- | --- |
| **CPU** | [Intel Core i5-11400T](https://www.cpubenchmark.net/cpu.php?id=4406&cpu=Intel+Core+i5-11400T+%40+1.30GHz) |
| **RAM** | 32 GB DDR4 |
| **Storage** | 2 TB NVMe SSD |

This setup is powerful enough to run multiple selfhosted services and virtual machines while still remaining reasonably power-efficient.

Without a wall power meter, total power draw is only an estimate. Software-side measurements show around **13-15 W** of CPU package power under light workloads, suggesting roughly **20-30 W** of total system power draw depending on configuration and workload.

### Hardware

Recommended hardware specifications:

| Component | Recommendation |
| --- | --- |
| **CPU** | A 4-core x86-64 CPU is a good baseline. Intel Core i3/i5 8th Gen or newer and comparable AMD CPUs are usually more than sufficient for a typical home server |
| **GPU** | Not required for selfhosting. Avoid a dedicated GPU unless you plan to experiment with [cloud gaming](https://en.wikipedia.org/wiki/Cloud_gaming), hardware transcoding, AI workloads, or GPU passthrough |
| **RAM** | 8 GB is a practical minimum. For multiple services or game servers, 16-32 GB is recommended |
| **Storage** | Preferably full SSD storage with 512 GB or more |

For most typical home server workloads, **64 GB of RAM is more than necessary**, but it may be useful for virtualization, game servers, databases, or other memory-intensive workloads.

For storage, an alternative to full SSD storage is to use a smaller **128-256 GB SSD for the operating system** combined with an HDD for larger data storage.

SSDs are faster, quieter, and more energy-efficient, but usually more expensive per gigabyte.

### Noise

Consider noise levels, especially if your server will be placed in a bedroom, office, or shared living space.

Mini PCs are usually quiet and power-efficient, while older desktops or enterprise servers may be noticeably louder. If needed, place the server in a less disruptive location.

A wired Ethernet connection is strongly recommended. Wi-Fi may work, but it can introduce lower speeds, higher latency, and reliability issues.

### Physical Size

Physical size also matters. Enterprise servers, repurposed desktops, or storage-heavy builds can take up a lot of space, especially when using multiple HDDs or RAID arrays.

Before choosing hardware, make sure you have enough space, decent ventilation, and easy access for maintenance.

### Recommended Steps

Before installing the operating system, hypervisor, or services:

- Update the BIOS or firmware.
- Check whether virtualization is enabled in BIOS/UEFI.
- Enable IOMMU/VT-d if you plan to use PCI or GPU passthrough.
- Decide where the server will be physically placed.
- Plan your storage layout.
- Use wired Ethernet whenever possible.
- Plan your local IP addressing and DNS.
- Decide which services should be public and which should remain available only through LAN/VPN.
- Decide which setup best matches your requirements:
  - **[Proxmox VE](proxmox)** for the current virtualized homelab architecture.
  - **[Port Forwarding](ports)** for a simpler bare-metal Docker server with direct network exposure.
  - **[Cloudflare Tunnels](tunnels)** if you prefer exposing web services without opening inbound ports.

To update your BIOS, search for `"BIOS download"` together with the name of your motherboard, mini PC, laptop, or prebuilt system.

### Requirements

The exact requirements depend on the setup you choose.

#### Common Requirements

- A machine where the server will run.
- Basic Linux command-line knowledge.
- Wired network access where possible.
- A domain name if you want to expose services publicly.

The examples in the **Docker-based setups** use **Cloudflare DNS**, so a Cloudflare account and a domain managed through Cloudflare are recommended for those setups.

#### Docker-based Setups

For the **[Cloudflare Tunnels](tunnels)** and **[Port Forwarding](ports)** setups, Docker must be installed on the server.

#### Proxmox VE

For the **[Proxmox VE](proxmox)** setup, Proxmox VE runs directly on bare metal, while application workloads are intended to run inside virtual machines or LXC containers rather than directly on the hypervisor.

The Proxmox management interface should remain available only from trusted LAN/VPN networks and should not be exposed directly to the Internet.

#### Remote Access Requirements

| Setup | Public IP required | Inbound ports required |
| --- | ---: | ---: |
| **Cloudflare Tunnels** | No | No |
| **Port Forwarding** | Yes | Yes |
| **Proxmox VE** | Yes | Yes |

For **Cloudflare Tunnels**, no public IP address or inbound port forwarding is required.

For **Port Forwarding**, you need a **public IP address** that is reachable from the Internet, and your connection must not be behind CGNAT.

The current **Proxmox VE** setup also requires a publicly reachable IP address because public services are exposed through the router to a reverse proxy running inside the virtualized environment. The Proxmox management interface itself remains private and is accessible only from trusted LAN/VPN networks.

For setup-specific instructions, see:

- **[Proxmox VE](proxmox)** - current setup
- **[Port Forwarding](ports)** - stable, complete, legacy setup
- **[Cloudflare Tunnels](tunnels)** - legacy setup

---

## Repository Structure

```text
.
├── tunnels/     # Legacy Cloudflare Tunnel setup
├── ports/       # Stable and complete legacy bare-metal setup
└── proxmox/     # Current Proxmox-based homelab
```

The **Port Forwarding** setup remains fully usable as a reference implementation of a traditional bare-metal home server, but it is no longer actively developed or updated.

The **Proxmox VE** setup is the current architecture and will receive future updates.

## Infrastructure as Code

Infrastructure as Code and automation for the Proxmox-based setup are maintained separately in **[PoProstuWitold/homelab-infra](https://github.com/PoProstuWitold/homelab-infra)**.

> [!NOTE]
> The infrastructure repository is currently a **work in progress**.
