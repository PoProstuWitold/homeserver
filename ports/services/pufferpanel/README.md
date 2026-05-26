# PufferPanel

Host and manage game servers using PufferPanel and Docker.

PufferPanel is a general-purpose game server management panel. It provides a web UI, file manager, SFTP access, console access, backups and server templates.

This setup uses PufferPanel as a replacement for the previous `itzg/minecraft-server` based setup.

The configuration below focuses on a Minecraft Java/Fabric server with optional Bedrock support through Geyser/Floodgate, but the same PufferPanel instance can also be used to manage other supported game servers.

---

## Requirements

- Docker
- Portainer
- A domain or subdomain, for example `mc.your-domain.tld`
- Free ports:

  - `8085/tcp` - PufferPanel Web UI
  - `5657/tcp` - PufferPanel SFTP
  - `25565/tcp` - Minecraft Java
  - `19132/udp` - Minecraft Bedrock/Geyser, if used

PufferPanel uses port `8080` for the web panel and `5657` for SFTP internally. In this setup, the web panel is exposed as `8085` on the host.

---

## Directory structure

```text
/srv/server/services/pufferpanel/
├── config/
└── data/
    └── servers/
```

Create required directories:

```bash
sudo mkdir -p /srv/server/services/pufferpanel/config
sudo mkdir -p /srv/server/services/pufferpanel/data
```

---

## Portainer Stack

Create a new stack in Portainer and use the following configuration:

`docker-compose.yml`

```yaml
services:
  pufferpanel:
    container_name: pufferpanel
    image: pufferpanel/pufferpanel:latest
    ports:
      - "8085:8080"
      - "5657:5657"
    volumes:
      - /srv/server/services/pufferpanel/config:/etc/pufferpanel
      - /srv/server/services/pufferpanel/data:/var/lib/pufferpanel
      - /var/run/docker.sock:/var/run/docker.sock
    restart: unless-stopped
```

---

## Create admin user

Create the first admin user inside the container:

```bash
docker exec -it pufferpanel /pufferpanel/bin/pufferpanel user add
```

Example:

```text
Username: YOUR_USERNAME
Email: YOUR_EMAIL
Password: **********************************
Confirm Password: **********************************
Set as Admin [y/N]: Yes
INFO  User added
```

> [!NOTE]
> Password must be between 8 and 72 characters.

Open the panel:

```text
http://YOUR_SERVER_IP:8085
```

---

## Minecraft server networking

For Minecraft servers created inside PufferPanel, use Docker host networking in the server advanced settings:

```text
Network Name: host
Port Bindings: empty
```

This allows the Minecraft server container to bind directly to host ports, for example:

```text
25565/tcp
19132/udp
```

Do not expose Minecraft ports in the PufferPanel stack if the Minecraft server uses `host` networking in PufferPanel.

Otherwise, the PufferPanel container may reserve these ports and the actual Minecraft server may fail to start.

---

## Creating a Fabric server

Create a new server in the PufferPanel Web UI:

1. Go to `Servers`.
2. Click `Create Server`.
3. Select or create a Fabric Docker template.
4. Use these example values:

```text
Server name: FABRIC_1_21_11
Minecraft version: 1.21.11
Fabric Loader: 0.19.2
Memory: 8192 MB
IP: 0.0.0.0
Port: 25565
Docker image: eclipse-temurin:21
Network Name: host
Port Bindings: empty
```

Start the server once to let PufferPanel generate all required files, then stop it before importing an existing server.

---

## Import existing Minecraft server

Example old server path:

```text
/srv/server/services/minecraft/servers/FABRIC_1_21_11
```

Example PufferPanel server path:

```text
/srv/server/services/pufferpanel/data/servers/<SERVER_ID>
```

Recommended files and folders to migrate:

```text
world/
mods/
config/
defaultconfigs/
libraries/
versions/

server.properties
server-icon.png
whitelist.json
ops.json
banned-ips.json
banned-players.json
usercache.json
eula.txt
```

Optional files and folders:

```text
logs/
crash-reports/
```

Usually not needed:

```text
.cache/
.install-modrinth.env
.rcon-cli.env
.rcon-cli.yaml
.modrinth-modpack-manifest.json
.fabric-manifest.json
```

If the old server has a Fabric launcher like:

```text
fabric-server-mc.1.21.11-loader.0.18.4-launcher.1.1.1.jar
```

rename it or replace the generated launcher so PufferPanel starts:

```text
fabric-server.jar
```

The server command should point to:

```bash
java -Xms1G -Xmx8192M -Dlog4j2.formatMsgNoLookups=true -jar fabric-server.jar nogui
```

---

## Updating Fabric Loader

To update Fabric Loader, download a new Fabric server launcher for the same Minecraft version and desired Fabric Loader version.

Example:

```text
fabric-server-mc.1.21.11-loader.0.19.2-launcher.1.1.1.jar
```

Then:

1. Stop the server in PufferPanel.
2. Upload the new launcher to the server root directory.
3. Rename the current launcher:

```text
fabric-server.jar -> fabric-server-0.18.4-backup.jar
```

4. Rename the new launcher:

```text
fabric-server-mc.1.21.11-loader.0.19.2-launcher.1.1.1.jar -> fabric-server.jar
```

5. Start the server.

Successful log example:

```text
Loading Minecraft 1.21.11 with Fabric Loader 0.19.2
Done (...)! For help, type "help"
```

Keep the old launcher backup for a few days in case some mod breaks on the newer loader.

---

## DNS records

For making your server accessible by `mc.your-domain.tld`, create these DNS records in Cloudflare or another DNS provider:

|Type|Name|Content|Proxy status|TTL|
|-|-|-|-|-|
|A|mc|YOUR_SERVER_IPV4|DNS Only|Auto|
|SRV|_minecraft._tcp.mc|0 0 25565 mc.your-domain.tld|DNS Only|Auto|

If you use the default Minecraft Java port `25565`, the SRV record is optional.

For Bedrock/Geyser, make sure UDP port `19132` is open on the server firewall.

---

## Check server status

You can verify the server status using:

```text
https://mcsrvstat.us/
```

Check:

```text
mc.your-domain.tld
```

Expected result:

```text
MOTD: Your server MOTD
Players: 0 / 32
Version: 1.21.11
```

---

## Notes

> [!IMPORTANT]
> Do not expose Minecraft ports in the PufferPanel stack if the Minecraft server uses `host` networking in PufferPanel.

> [!TIP]
> Keep the old `itzg/minecraft-server` directory and exported ZIP backup for a few days after migration.

> [!NOTE]
> A single `Can't keep up!` warning after startup is usually normal for a heavily modded server. 
>
> If it appears constantly during gameplay, reduce `view-distance`, check TPS with Spark, or tune JVM flags.

