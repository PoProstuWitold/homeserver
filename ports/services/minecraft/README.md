# Minecraft
Host your favorite game and make it accesible.

I will use this [dockerized Minecraft](https://github.com/itzg/docker-minecraft-server). It has many [configuration](https://docker-minecraft-server.readthedocs.io/en/latest/variables/) options such as changing server engine (e.g. Forge, Fabric, Vanilla, Spigot).

I also included [RCON web admin console](https://github.com/itzg/docker-rcon-web-admin) for easier server managment for server's operators. If you don't want this feature simple delete ``rcon`` service below.

``docker-compose.yml``
```yaml
services:
  rcon:
    container_name: rcon
    image: itzg/rcon
    ports:
      - 4326:4326
      - 4327:4327
    environment:
      RWA_USERNAME: "admin"
      RWA_PASSWORD: "changeme"
      RWA_ADMIN: true
      RWA_SERVER_NAME: "mc.your-domain.tld"
      RWA_RCON_HOST: mc
      RWA_RCON_PASSWORD: "changeme_rcon" # must be the same as in Minecraft server
    depends_on:
    - mc # we need to be sure that RCON web admin will be started after Minecraft server
    restart: unless-stopped
    volumes:
      - /srv/server/services/rcon:/opt/rcon-web-admin/db
  mc:
    container_name: mc
    image: itzg/minecraft-server
    ports:
      - "25565:25565"
      - "19132:19132/udp"
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
      # ICON: "https://your-domain.tld/imgs/mc_icon.png"
      # OVERRIDE_ICON: true
      MAX_PLAYERS: 32
      ENABLE_COMMAND_BLOCK: true
      VIEW_DISTANCE: 16
      SERVER_NAME: "mc.your-domain.tld"
      EXEC_DIRECTLY: true
      ENABLE_QUERY: true
      # WHITELIST
      ENABLE_WHITELIST: true
      WHITELIST: "Steve,Alex"
      # RCON
      ENABLE_RCON: true
      RCON_PASSWORD: "changeme_rcon" # must be the same as in RCON web admin
	  # Mods
      MOD_PLATFORM: "MODRINTH"
      MODRINTH_MODPACK: "modpack that icludes Geyser and Floodgate"
    restart: unless-stopped
    volumes:
      - /srv/server/services/minecraft/servers/FABRIC_1_21:/data
```

For making your server accessible by ``mc.your-domain.tld`` you need to follow these steps:

1. Create following DNS records in your Cloudflare dashboard:

| **Type** 	|      **Name**      	|          **Content**         	| **Proxy status** 	| **TTL** 	|
|:--------:	|:------------------:	|:----------------------------:	|:----------------:	|:-------:	|
|     A    	|         mc         	|       YOUR_SERVER_IPV4       	|     DNS Only     	|   Auto  	|
|    SRV   	| _minecraft._tcp.mc 	| 0 0 25565 mc.your-domain.tld 	|     DNS Only     	|   Auto  	|

If you have **Cloudflare DDNS** already running, the A record should be already in place.

You can change name and instead ``mc`` type ``play`` or any other string value you want.

You don't need the SRV record if you are using a default Minecraft port (25565), but it will be more future-proof if you decide to eventually change the default port.

2. Ensure everything is working by checking it on [mcsrvstat.us](https://mcsrvstat.us/)