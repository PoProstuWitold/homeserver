# qBittorrent & Collections

Everything needed for automating media pulling to **Jellyfin** through **Jellyseerr**. I decided to group everything together because it's easier to manage and all these services have great synergy together.

Of course you can modify it as you wish. Just preserve volume levels for ``data`` and ``downloads`` so you ***Starr Apps*** will correctly resolve your media.

Mandatory:
- **[qBittorrent](https://github.com/linuxserver/docker-qbittorrent)** - download client.
- **[Prowlarr](https://github.com/linuxserver/docker-prowlarr)** - indexer manager/proxy for collections.
- **[Flaresolverr](https://github.com/FlareSolverr/FlareSolverr)** - proxy server to bypass Cloudflare protection.

Optional:
- **[Radarr](https://github.com/linuxserver/docker-radarr)** - movie collection manager.
- **[Sonarr](https://github.com/linuxserver/docker-sonarr)** - tv shows/anime collection manager.
- **[Lidarr](https://github.com/linuxserver/docker-lidarr)** - music collection manager.
- **[Readarr](https://github.com/linuxserver/docker-readarr)** - ebook and audiobook collection manager.

``docker-compose.yml``
```yaml
services:
  qbittorrent: # download client
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
      - /srv/server/media/downloads:/downloads
    restart: unless-stopped
  prowlarr: # indexer manager/proxy for collections
    image: lscr.io/linuxserver/prowlarr:latest
    container_name: prowlarr
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Warsaw
    volumes:
      - /srv/server/services/prowlarr/config:/config
    ports:
      - 9696:9696
    restart: unless-stopped
  radarr: # movie collection manager
    image: lscr.io/linuxserver/radarr:latest
    container_name: radarr
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Warsaw
    volumes:
      - /srv/server/services/radarr/config:/config
      - /srv/server/media:/data
      - /srv/server/media/downloads:/downloads
    ports:
      - 7878:7878
    restart: unless-stopped
  sonarr: # tv shows/anime collection manager
    image: lscr.io/linuxserver/sonarr:latest
    container_name: sonarr
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Warsaw
    volumes:
      - /srv/server/services/sonarr/config:/config
      - /srv/server/media:/data
      - /srv/server/media/downloads:/downloads
    ports:
      - 8989:8989
    restart: unless-stopped
  lidarr: # music collection manager
    image: lscr.io/linuxserver/lidarr:latest
    container_name: lidarr
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Warsaw
    volumes:
      - /srv/server/services/lidarr/config:/config
      - /srv/server/media:/data
      - /srv/server/media/downloads:/downloads
    ports:
      - 8686:8686
    restart: unless-stopped
  readarr: # ebook and audiobook collection manager
    image: lscr.io/linuxserver/readarr:develop
    container_name: readarr
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Warsaw
    volumes:
      - /srv/server/services/readarr/config:/config
      - /srv/server/media:/data
      - /srv/server/media/downloads:/downloads
    ports:
      - 8787:8787
    restart: unless-stopped
  flaresolverr: # Proxy server to bypass Cloudflare protection
    image: ghcr.io/flaresolverr/flaresolverr:latest
    container_name: flaresolverr
    environment:
      - TZ=Europe/Warsaw
      - LANG=en_US
    ports:
      - 8191:8191
    restart: unless-stopped
```