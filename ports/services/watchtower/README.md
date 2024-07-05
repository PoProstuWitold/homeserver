# Watchtower
A process for automating Docker container base image updates.

You can disable watchtower for particular services by adding label to their respective containers:

``com.centurylinklabs.watchtower.enable = false``

Or made them ``monitor-only``:

``com.centurylinklabs.watchtower.monitor-only=true``

> **IMPORTANT!** I higly recommend setting critical services such as ``caddy`` and ``wg_easy`` to ``monitor-only=true``

``docker-compose.yml``
```yaml
services:
  watchtower:
    image: containrrr/watchtower:latest
    container_name: watchtower
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      - WATCHTOWER_CLEANUP=true
      - WATCHTOWER_LOG_FORMAT=Pretty
      - WATCHTOWER_INCLUDE_RESTARTING=true
      - WATCHTOWER_INCLUDE_STOPPED=true
      - WATCHTOWER_REVIVE_STOPPED=false
      - WATCHTOWER_TIMEOUT=30s
      - WATCHTOWER_SCHEDULE=0 0 1 * * *
      - TZ=Europe/Warsaw
    network_mode: bridge
    restart: unless-stopped
```

With this setup cron job to update all containers will run everyday at 1:00 of specified timezone.