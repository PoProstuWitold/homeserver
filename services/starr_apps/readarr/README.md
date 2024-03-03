# Readarr

**WARNING!** 
Readarr is currently in beta testing and is generally still in a work in progress. Features may be broken, incomplete, or cause spontaneous combustion

[Readarr](https://github.com/Readarr/Readarr) is an ebook and audiobook collection manager for Usenet and BitTorrent users. It can monitor multiple RSS feeds for new books from your favorite authors and will grab, sort, and rename them.

``docker-compose.yml``
```yaml
version: "3.8"

services:
  readarr:
    image: lscr.io/linuxserver/readarr:develop
    container_name: readarr
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Warsaw
    volumes:
      - /home/docker/readarr/config:/config
      - /home/docker/jellyfin/media:/data
    ports:
      - 8787:8787
    restart: unless-stopped
```