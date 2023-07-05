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