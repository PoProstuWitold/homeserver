# Watchtower
A process for automating Docker container base image updates.

**Important note:** with configuration provided below ``Watchtower`` updates all images of all running containers. It is not recommended to use it with tag *latest* or/and for critical services as in case of breaking changes some of your services may **break** . Use with caution.

You can disable watchtower for particular services by adding label to their respective containers:

``com.centurylinklabs.watchtower.enable = false``

Or made them ``monitor-only``:

``com.centurylinklabs.watchtower.monitor-only=true``

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
      - WATCHTOWER_SCHEDULE=0 0 19 * * *
      - TZ=Europe/Warsaw
    network_mode: bridge
    restart: unless-stopped
```

With this setup cron job to update all containers will run everyday at 19:00 of specified timezone.