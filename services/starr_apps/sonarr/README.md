# Sonarr
[Sonarr](https://github.com/Sonarr/Sonarr) is a PVR for Usenet and BitTorrent users. It can monitor multiple RSS feeds for new episodes of your favorite shows and will grab, sort and rename them.

``docker-compose.yml``
```yaml
version: "3.8"

services:
  sonarr:
    image: lscr.io/linuxserver/sonarr:latest
    container_name: sonarr
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Warsaw
    volumes:
      - /home/docker/sonarr/config:/config
      - /home/docker/jellyfin/media:/data
    ports:
      - 8989:8989
    restart: unless-stopped
```