# Homeserver

> Link to article [here](https://witoldzawada.dev/blog/introduction-to-selfhosting)

In this guide, you'll find easy-to-follow steps to set up your own server at home using ***Linux***, ***Docker***, and either ***[Port Forwarding](ports)*** or ***[Cloudflare Tunnels](tunnels)***. 

Please note that some parts of this guide are based on my personal preferences (e.g. the choice of Linux distros), and your setup for certain things may slightly differ. Let's begin, shall we?

> **IMPORTANT!** I'll try to keep this guide free of any costs besides: power consumption, domain renewal and one-time purchases such as a router, therefore some things such as VPS or any subscription-based services may not be covered (or at least are not planned for now).


## 0. Things to Consider: Power Consumption, Hardware, Noise, Physical Size, Recommended Steps & Requirements

### Power Consumption
Since your server will likely be running 24/7, it is important to consider both power efficiency and noise levels. You can use an older PC, a **[Raspberry Pi 5](https://www.raspberrypi.com/products/raspberry-pi-5/)**, or a compact business-class mini PC (commonly used models include **Lenovo ThinkCentre**, **Dell OptiPlex**, or **HP EliteDesk**).

In this guide, I will be using a **[Lenovo ThinkCentre M70q Gen 2](https://pcsupport.lenovo.com/us/en/products/desktops-and-all-in-ones/thinkcentre-m-series-desktops/thinkcentre-m70q-gen-2/documentation/?linkTrack=footer%3ASupport_Manuals)** equipped with an [Intel Core i5-11400T](https://www.cpubenchmark.net/cpu.php?id=4406&cpu=Intel+Core+i5-11400T+%40+1.30GHz) processor (35W TDP), **32 GB DDR4 memory**, and a **2 TB NVMe SSD**.  
This configuration provides an excellent balance of performance and energy efficiency, with typical consumption of ~30-40W under light workloads, making it a reliable and cost-effective choice for continuous operation.

### Hardware
Recommended hardware specifications:
- **CPU**: at least Intel Core i3/i5 9th Gen (or AMD equivalent) with 4+ cores.
- **GPU**: not required for selfhosting; a dedicated GPU will only increase power consumption unless you plan to experiment with ***[cloud gaming](https://en.wikipedia.org/wiki/Cloud_gaming)***.
- **RAM**: minimum 8GB. For running multiple services or game servers, 16-32GB may be required. In most cases, 64 GB is excessive, but viable if budget allows.
- **Storage**: ideally full SSD (512GB or more). Alternatively, use an SSD for the OS (128–256 GB) combined with an HDD (512GB or more) for data. SSDs are faster and more energy efficient, though more expensive.

### Noise
Consider noise levels, especially if your server will be placed in shared living spaces. If necessary, relocate it to a less disruptive location. Always ensure a stable Ethernet connection to your router; relying on Wi-Fi can cause significant bandwidth loss.

### Physical Size
Keep in mind that enterprise servers or repurposed desktops may be bulky, especially when configured with multiple HDDs in RAID arrays. Ensure you have adequate space and ventilation before setting up.

### Recommended Steps
Although not strictly required, it is recommended to update your BIOS before starting. Search for "BIOS download" along with the name of your motherboard or device (e.g. prebuilt PC, laptop) to find the latest version.

### Requirements
Regardless of whether you choose ***Cloudflare Tunnels*** or ***Port Forwarding***, you will need:
- Your own domain (either purchased via Cloudflare or transferred to their DNS servers).
- A Cloudflare account.
- A workstation where the server will run.

For ***Cloudflare Tunnels***, no further requirements apply.  
For ***Port Forwarding***, you will additionally need a static or dynamic **public IP**.

For detailed instructions, see:
- **[Cloudflare Tunnels](tunnels)**
- **[Port Forwarding](ports)**