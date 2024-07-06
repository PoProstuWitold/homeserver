# Homarr
[Homarr](https://github.com/ajnart/homarr) is a customizable browser's home page to interact with your homeserver's Docker containers (e.g. Sonarr/Radarr).

Homarr features [integrations](https://homarr.dev/docs/tags/integration/) with many services included in this repository such as:
- Dashdot


``docker-compose.yml``
```yaml
services:
  homarr:
    container_name: homarr
    image: ghcr.io/ajnart/homarr:latest
    ports:
      - 7575:7575
    networks:
      - caddy
    volumes:
      # - /var/run/docker.sock:/var/run/docker.sock # Optional, only if you want docker integration
      - /srv/server/services/homarr/configs:/app/data/configs
      - /srv/server/services/homarr/icons:/app/public/icons
      - /srv/server/services/homarr/images:/app/public/imgs
      - /srv/server/services/homarr/data:/data
    restart: unless-stopped

networks:
  caddy:
    name: caddy
    external: true
```