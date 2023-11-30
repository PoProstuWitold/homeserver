# Watchtower
A process for automating Docker container base image updates.

``docker-compose.yml``
```yaml
version: "3.8"

services:
  watchtower:
    image: containrrr/watchtower:latest
    container_name: watchtower
    restart: unless-stopped
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      - WATCHTOWER_CLEANUP=true
      - WATCHTOWER_INCLUDE_RESTARTING=true
      - WATCHTOWER_INCLUDE_STOPPED=true
      - WATCHTOWER_REVIVE_STOPPED=false
      - WATCHTOWER_NO_RESTART=false
      - WATCHTOWER_TIMEOUT=30s
      - WATCHTOWER_SCHEDULE=0 0 19 * * *
      - WATCHTOWER_DEBUG=false
      - TZ=Europe/Warsaw
    network_mode: bridge
```

With this setup cron job to update all containers will run everyday at 19:00 of specified timezone.