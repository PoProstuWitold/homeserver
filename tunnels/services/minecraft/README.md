# Minecraft
Host your favorite game and make it accesible.

I will use this [dockerized Minecraft](https://github.com/itzg/docker-minecraft-server). It has many [configuration](https://docker-minecraft-server.readthedocs.io/en/latest/variables/) options such as changing server engine (e.g. Forge, Fabric, Vanilla, Spigot)

``docker-compose.yml``
```yaml
services:
  mc:
    container_name: mc
    image: itzg/minecraft-server:latest
    ports:
      - 25565:25565
    tty: true
    stdin_open: true
    environment:
      # GENERAL
      MEMORY: "6G"
      MAX_MEMORY: "12G"
      TZ: "Europe/Warsaw"
      LOG_TIMESTAMP: true
      # SERVER
      TYPE: "FABRIC"
      EULA: true
      VERSION: "1.21"
      MOTD: "Motto of the day"
      DIFFICULTY: "normal"
      # ICON: "url"
      # OVERRIDE_ICON: true
      MAX_PLAYERS: 32
      ENABLE_COMMAND_BLOCK: true
      VIEW_DISTANCE: 16
      SERVER_NAME: "server name"
      EXEC_DIRECTLY: true
      ENABLE_QUERY: true
      # WHITELIST
      ENABLE_WHITELIST: true
      WHITELIST: "Steve,Alex"
    restart: unless-stopped
    volumes:
      - /srv/server/services/minecraft/servers/FABRIC_1_21:/data
```

When your setup is working and you can connect to your server via ``your-ipv4:25565`` and want to make your server accesible via ``mc.your-domain.tld`` then follow this guide.
### 1. Setup ``playit.gg`` and copy your server IP

Go to [playit.gg](https://playit.gg/), download their software, make TCP Minecraft Java tunnel.

Next, under the ``Tunnels`` tab you will see your tunnel name and its ip (something randomly generated like: ``camera-cat.joinmc.link``). Copy it.

### 2. Get your server DNS info and make similar records in Cloudflare
Paste previously copied IP to [mcsrvstat](https://mcsrvstat.us/). Scroll down and click ``Show DNS info``. Now you need to create two similar (AAAA and SRV) records under your domain in cloudflare.

It should be fairly simple, but if you need additional help this [post](https://discuss.playit.gg/t/custom-free-domain-for-your-server-freenom/461) may help you.

In the end you should be able to join to your server via ``mc.your-domain.tld``.