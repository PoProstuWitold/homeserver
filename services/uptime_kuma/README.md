# Uptime Kuma
This software lets you make healthchecks with many protocols, databases, docker containers and many more. You can also combine them and integrate into healthcheck url that may look like: ``https://uptime.YOURDOMAIN.com/status/jellyfin``

``docker-compose.yml``
```yaml
version: "3.8"

services:
  uptime-kuma:
    image: louislam/uptime-kuma:1
    container_name: uptime-kuma
    restart: always
    ports:
      - "3001:3001"
    volumes:
      - uptime-kuma:/app/data
      - /var/run/docker.sock:/var/run/docker.sock

volumes:
  uptime-kuma:

```
