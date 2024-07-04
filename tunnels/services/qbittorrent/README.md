# qBittorrent
[qBittorrent](https://github.com/qbittorrent/qBittorrent) BitTorrent client

``docker-compose.yml``
```yaml
services:
  qbittorrent:
    image: lscr.io/linuxserver/qbittorrent:latest
    container_name: qbittorrent
    ports:
      - 8084:8084
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Warsaw
      - WEBUI_PORT=8084
    volumes:
      - /srv/server/services/qbittorrent/config:/config
      - /srv/server/media/downloads:/data/downloads
    restart: unless-stopped
```