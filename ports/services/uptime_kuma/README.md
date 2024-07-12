# Uptime Kuma
This software lets you make healthchecks with many protocols, databases, docker containers and many more. 

You can also combine them and integrate into healthcheck url that may look like: ``https://uptime.your-domain.tld/status/jellyfin``

``docker-compose.yml``
```yaml
services:
  uptime_kuma:
    image: louislam/uptime-kuma:latest
    container_name: uptime_kuma
    ports:
      - 3002:3001
    networks:
      - caddy
    volumes:
      - /srv/server/services/uptime_kuma:/app/data
      - /var/run/docker.sock:/var/run/docker.sock
    restart: unless-stopped

networks:
  caddy:
    name: caddy
    external: true
```