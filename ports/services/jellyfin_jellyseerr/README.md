# Jellyfin & Jellyseerr

**[Jellyfin](https://github.com/jellyfin/jellyfin)** a free software media system that puts you in control of managing and streaming your media. It is an alternative to the proprietary Emby and Plex, to provide media from a dedicated server to end-user devices via multiple apps.

**[Jellyseerr](https://github.com/Fallenbagel/jellyseerr)** is a free and open source software application for managing requests for your media library. It is a fork of Overseerr built to bring support for Jellyfin & Emby media servers!

You need to create your library folders inside ``/srv/server/media``.

``docker-compose.yml``
```yaml
services:
  jellyfin:
    image: lscr.io/linuxserver/jellyfin:latest
    container_name: jellyfin
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Warsaw
      - JELLYFIN_PublishedServerUrl=jellyfin.your-domain.tld
    volumes:
      - /srv/server/services/jellyfin/config:/config
      - /srv/server/media:/data
    ports:
      - 8096:8096
    networks:
      - caddy
    restart: unless-stopped
  jellyseerr:
    image: fallenbagel/jellyseerr:latest
    container_name: jellyseerr
    ports:
      - 5055:5055
    networks:
      - caddy
    environment:
      - LOG_LEVEL=debug
      - TZ=Europe/Warsaw
    volumes:
      - /srv/server/services/jellyseerr/config:/app/config
    restart: unless-stopped

networks:
  caddy:
    name: caddy
    external: true
```