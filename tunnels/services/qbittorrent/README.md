# qBittorrent
[qBittorrent](https://github.com/qbittorrent/qBittorrent) BitTorrent client

``docker-compose.yml``
```yaml
version: "3.8"

services:
  qbittorrent:
    image: lscr.io/linuxserver/qbittorrent:latest
    container_name: qbittorrent
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Warsaw
      - WEBUI_PORT=8084
    volumes:
      - /home/docker/qbittorrent/config:/config
      - /home/docker/jellyfin/media/downloads:/data/downloads
    ports:
      - 8084:8084
    restart: unless-stopped
```