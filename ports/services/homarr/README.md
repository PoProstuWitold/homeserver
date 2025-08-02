# Homarr
[Homarr](https://github.com/ajnart/homarr) is a customizable browser's home page to interact with your homeserver's Docker containers (e.g. Sonarr/Radarr).

You can get ``SECRET_ENCRYPTION_KEY`` by following official **[Homarr Docker Installation Guide](https://homarr.dev/docs/getting-started/installation/docker)**.

``docker-compose.yml``
```yaml
services:
  homarr:
    container_name: homarr
    image: ghcr.io/homarr-labs/homarr:latest
    ports:
      - 7575:7575
    networks:
      - caddy
    volumes:
      # - /var/run/docker.sock:/var/run/docker.sock # Optional, only if you want docker integration
      - /srv/server/services/homarr/appdata:/appdata
    environment:
      - SECRET_ENCRYPTION_KEY=
    restart: unless-stopped

networks:
  caddy:
    name: caddy
    external: true
```