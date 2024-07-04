# Prowlarr
[Prowlarr](https://github.com/Prowlarr/Prowlarr) is an indexer manager/proxy built on the popular *arr .net/reactjs base stack to integrate with your various PVR apps.

``docker-compose.yml``
```yaml
services:
  prowlarr:
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
```