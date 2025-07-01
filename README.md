# Homeserver

> Link to article [here](https://witoldzawada.dev/blog/introduction-to-selfhosting)

In this guide, you’ll find easy-to-follow steps to set up your own server at home using ***Linux***, ***Docker***, and either ***[Port Forwarding](ports)*** or ***[Cloudflare Tunnels](tunnels)***. 

Please note that some parts of this guide are based on my personal preferences (e.g. the choice of Linux distros), and your setup for certain things may slightly differ. Let’s begin, shall we?

> **IMPORTANT!** I'll try to keep this guide free of any costs besides: power consumption, domain reneval and one-time purchases such as a router, therefore some things such as VPS or any subscription-based services may not be covered (or at least are not planned for now).


## 0. Things to Consider: Power Consumption, Hardware, Noise, Physical Size, Recommended Steps & Requirements

### Power Consumption
Remember that your server is likely going to run 24/7, so keep in mind the energy consumption of your workstation and its noise. You can use your old PC, a [Raspberry Pi 5](https://www.raspberrypi.com/products/raspberry-pi-5/), or some mini PC (I recommend older, used HP, Dell, Lenovo, or Intel NUC models). In this guide, I will be using an **[Intel NUC11TNHI5](https://www.intel.com/content/www/us/en/products/sku/205594/intel-nuc-11-pro-kit-nuc11tnhi5/specifications.html)** with 32GB RAM and a 2TB SSD, as it only consumes 28W of energy. It’s not necessary to buy exactly the same hardware as mine to follow this tutorial.


### Hardware
In terms of hardware, here are my recommendations:
- **CPU**: at least 9th generation Intel Core i3 or i5 or AMD equivalent; 4+ cores.
- **GPU**: don't run the server with a gpu (unless you want your own ***[gaming cloud](https://en.wikipedia.org/wiki/Cloud_gaming)***) as you won't need it and it will greatly increase power consumption.
- **RAM**: I recommend a minimum of 8GB. If you're going to run lots of services, then 16GB or even 32GB may be necessary, especially if you want to run game servers. In 90% of cases, 64GB is overkill, but if you can afford it and want it, then go ahead.
- **STORAGE**: I recommend either going full SSD (at least 512GB) or using an SSD for the OS (128GB or 256GB) and an HDD (min. 512GB) for data. SSDs are more energy-efficient but also more expensive.


### Noise
Your home server might be *a bit* noisy, especially for your family or roommates. Make sure its noise level is acceptable to them or, if it isn't, consider relocating it to a less disruptive spot. If you move your server further away from your main router, ensure it remains directly connected via an Ethernet cable. Relying on Wi-Fi could result in significant bandwidth loss.


### Physical Size
Whether you build your server, buy an enterprise model, or repurpose a regular PC for self-hosting, be mindful of its physical size. These machines often house multiple HDDs connected to some form of RAID array, which can make them quite bulky. Keep this in mind when choosing an appropriate location for your server.


### Recommended Steps
Although it is not required I recommend updating your BIOS. Just google "bios download" and name of your motherboard or name of your device (e.g. premade PC, laptop).


### Requirements
In case of both going with ***Cloudflare Tunnels*** or ***Port Forwarding*** you will need:
- Your own domain (you have to either buy it through Cloudflare or move your existing domain to their DNS servers).
- A Cloudflare account.
- A workstation where the server will be running.

With ***Cloudflare Tunnels*** there are no further requirements but in case of ***Port Forwarding*** you will also need static or dynamic **public** IP.

For more specific instructions go to one of below:
- **[Cloudflare Tunnels](tunnels)**
- **[Port Forwarding](ports)**