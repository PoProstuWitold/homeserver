# Uptime Kuma
This software lets you make healthchecks with many protocols, databases, docker containers and many more. 

You can also combine them and integrate into healthcheck url that may look like: ``https://uptime.your-domain.tld/status/jellyfin``

``docker-compose.yml``
```yaml
services:
  uptime_kuma:
    image: louislam/uptime-kuma:1
    container_name: uptime_kuma
    ports:
      - 3001:3001
    volumes:
      - /srv/server/services/uptime_kuma:/app/data
      - /var/run/docker.sock:/var/run/docker.sock
    restart: unless-stopped
```