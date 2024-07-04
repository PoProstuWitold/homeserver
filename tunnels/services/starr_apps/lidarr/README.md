# Lidarr
[Lidarr](https://github.com/Lidarr/Lidarr) is a music collection manager for Usenet and BitTorrent users. It can monitor multiple RSS feeds for new tracks from your favorite artists and will grab, sort and rename them.

``docker-compose.yml``
```yaml
services:
  lidarr:
    image: lscr.io/linuxserver/lidarr:latest
    container_name: lidarr
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Warsaw
    volumes:
      - /srv/server/services/lidarr/config:/config
      - /srv/server/media:/data
    ports:
      - 8686:8686
    restart: unless-stopped
```