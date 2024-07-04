# Radarr
[Radarr](https://github.com/Radarr/Radarr) is a movie collection manager for Usenet and BitTorrent users. It can monitor multiple RSS feeds for new movies and will interface with clients and indexers to grab, sort, and rename them.

``docker-compose.yml``
```yaml
services:
  radarr:
    image: lscr.io/linuxserver/radarr:latest
    container_name: radarr
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Warsaw
    volumes:
      - /srv/server/services/radarr/config:/config
      - /srv/server/media/media:/data
    ports:
      - 7878:7878
    restart: unless-stopped
```