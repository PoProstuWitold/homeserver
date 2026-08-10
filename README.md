# Homeserver

> [!NOTE]
> For more background and context, you can also read my [article about selfhosting](https://witoldzawada.dev/blog/introduction-to-selfhosting).

This guide walks through building and maintaining a home server using ***Linux***, ***Docker***, and either ***[Port Forwarding](ports)*** or ***[Cloudflare Tunnels](tunnels)*** for remote access.

Some parts of this guide reflect my personal preferences, such as Linux distributions, hardware, directory structure, and services. Your setup may differ depending on your hardware, network, and use case.

> [!IMPORTANT]
> This guide is intended to stay free of recurring costs, except for power consumption, domain renewal, and one-time hardware purchases such as a router or server device. 
>
> For this reason, VPS-based setups and subscription-based services are not covered, or at least are not planned for now.

---

## 0. Things to Consider Before Starting

Before setting up a home server, it is worth planning the hardware, power usage, noise level, physical placement, network access, and basic requirements.

### Power Consumption

A home server usually runs 24/7, so power efficiency matters from the beginning.

You can use an older PC, a **[Raspberry Pi 5](https://www.raspberrypi.com/products/raspberry-pi-5/)**, or a compact business-class mini PC. Commonly used models include **Lenovo ThinkCentre**, **Dell OptiPlex**, and **HP EliteDesk**.

The reference system used throughout this guide is a **[Lenovo ThinkCentre M70q Gen 2](https://pcsupport.lenovo.com/us/en/products/desktops-and-all-in-ones/thinkcentre-m-series-desktops/thinkcentre-m70q-gen-2/documentation/?linkTrack=footer%3ASupport_Manuals)** equipped with an [Intel Core i5-11400T](https://www.cpubenchmark.net/cpu.php?id=4406&cpu=Intel+Core+i5-11400T+%40+1.30GHz) processor, **32 GB DDR4 memory**, and a **2 TB NVMe SSD**.

This setup is powerful enough to run multiple selfhosted services while still staying reasonably power-efficient. Without a wall power meter, total power draw is only an estimate; software-side measurements show around **13–15 W** CPU package power under light workloads, suggesting roughly **20–30 W** total system power draw depending on configuration and workload.

### Hardware

Recommended hardware specifications:

- **CPU**: a 4-core x86-64 CPU is a good baseline. Intel Core i3/i5 8th-9th Gen or newer and comparable AMD CPUs are usually more than sufficient for a typical home server.
- **GPU**: not required for selfhosting. Avoid a dedicated GPU unless you plan to experiment with ***[cloud gaming](https://en.wikipedia.org/wiki/Cloud_gaming)***, hardware transcoding, or AI workloads, as it will increase power consumption.
- **RAM**: 8 GB is a practical minimum. For running multiple services or game servers, 16-32 GB is recommended. For most typical home server workloads, 64 GB is more than necessary, but it may be useful for virtualization, game servers, databases, or other memory-intensive workloads.
- **Storage**: preferably full SSD storage, 512 GB or more. Alternatively, use an SSD for the operating system, for example 128-256 GB, combined with an HDD for larger data storage. SSDs are faster, quieter, and more energy efficient, but usually more expensive per gigabyte.

### Noise

Consider noise levels, especially if your server will be placed in a bedroom, office, or shared living space.

Mini PCs are usually quiet and power-efficient, while older desktops or enterprise servers may be noticeably louder. If needed, place the server in a less disruptive location.

A wired Ethernet connection is strongly recommended. Wi-Fi may work, but it can introduce lower speeds, higher latency, and reliability issues.

### Physical Size

Physical size also matters. Enterprise servers, repurposed desktops, or storage-heavy builds can take up a lot of space, especially when using multiple HDDs or RAID arrays.

Before choosing hardware, make sure you have enough space, decent ventilation, and easy access for maintenance.

### Recommended Steps

Before installing the operating system and services:

- Update the BIOS or firmware.
- Check whether virtualization is enabled in BIOS/UEFI.
- Decide where the server will be physically placed.
- Plan your storage layout.
- Use wired Ethernet whenever possible.
- Decide whether you want to use ***Port Forwarding*** or ***Cloudflare Tunnels***.

To update your BIOS, search for `"BIOS download"` together with the name of your motherboard, mini PC, laptop, or prebuilt system.

### Requirements

This guide assumes the following baseline setup:

- A machine where the server will run.
- Basic Linux command-line knowledge.
- Docker installed on the server.
- A domain name if you want to expose services publicly.

The examples in this guide use **Cloudflare DNS**, so a Cloudflare account and a domain managed through Cloudflare are recommended.

For ***Cloudflare Tunnels***, no public IP is required.

For ***Port Forwarding***, you will additionally need a **public IP address** that is reachable from the Internet, and your connection must not be behind CGNAT.

For setup-specific instructions, see:

- **[Cloudflare Tunnels](tunnels)**
- **[Port Forwarding](ports)**