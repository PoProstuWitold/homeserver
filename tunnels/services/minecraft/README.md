# Minecraft
Host your favorite game and make it accesible.

I will use this [dockerized Minecraft](https://github.com/itzg/docker-minecraft-server). It has many [configuration](https://docker-minecraft-server.readthedocs.io/en/latest/variables/) options such as changing server engine (e.g. Forge, Fabric, Vanilla, Spigot)

``docker-compose.yml``
```yaml
version: "3.8"

services:
  mc:
    image: itzg/minecraft-server
    ports:
      - "25565:25565"
    tty: true
    stdin_open: true
    environment:
      # OS OPTIONS
      MEMORY: "4G"
      MAX_MEMORY: "12G"
      TZ: "Europe/Warsaw"
      # SERVER OPTIONS
      EULA: "TRUE"
      SERVER_NAME: "Homster Server"
      TYPE: "FABRIC"
      VERSION: "1.19.2"
      MOTD: "Minecraft server for Homsters and their owners"
      DIFFICULTY: "normal"
      # ICON: "paste url"
      # OVERRIDE_ICON: "TRUE"
      MAX_PLAYERS: 10
      ENABLE_COMMAND_BLOCK: "TRUE"
      VIEW_DISTANCE: 15
	  # you can also add WHITELIST and so on
    restart: unless-stopped
    volumes:
      - /home/docker/minecraft:/data
```

When your setup is working and you can connect to your server via ``your-ipv4:25565`` and want to make your server accesible via ``mc.your-domain.com`` then follow this guide.
### 1. Setup ``playit.gg`` and copy your server IP

Go to [playit.gg](https://playit.gg/), download their software, make TCP Minecraft Java tunnel.

Next, under the ``Tunnels`` tab you will see your tunnel name and its ip (something randomly generated like: ``camera-cat.joinmc.link``). Copy it.

### 2. Get your server DNS info and make similar records in Cloudflare
Paste previously copied IP to [mcsrvstat](https://mcsrvstat.us/). Scroll down and click ``Show DNS info``. Now you need to create two similar (AAAA and SRV) records under your domain in cloudflare.

It should be fairly simple, but if you need additional help this [post](https://discuss.playit.gg/t/custom-free-domain-for-your-server-freenom/461) may help you.

In the end you should be able to join to your server via ``mc.your-domain.com``.


