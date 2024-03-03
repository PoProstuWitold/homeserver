# Homarr
[Homarr](https://github.com/ajnart/homarr) is a customizable browser's home page to interact with your homeserver's Docker containers (e.g. Sonarr/Radarr).

Homarr features [integrations](https://homarr.dev/docs/tags/integration/) with many services included in this repository such as:
- Torrent (qBittorrent)
- Media Server (Jellyfin)
- Home Assistant
- Calendar (upcoming content from Starr Apps)
- Dash.
- Indexer Manager (Prowlarr)
- Download Speed (qBittorrent)


``docker-compose.yml``
```yaml
version: "3.8"

services:
  homarr:
    container_name: homarr
    image: ghcr.io/ajnart/homarr:latest
    restart: unless-stopped
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock # Optional, only if you want docker integration
      - /home/docker/homarr/configs:/app/data/configs
      - /home/docker/homarr/icons:/app/public/icons
      - /home/docker/homarr/data:/data
    ports:
      - '7575:7575'
```