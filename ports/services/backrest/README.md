# Backrest
[Backrest](https://github.com/garethgeorge/backrest) is a web UI and orchestrator for [restic](https://restic.net/) backup.

``docker-compose.yml``
```yaml
services:
  backrest:
    image: garethgeorge/backrest:latest
    container_name: backrest
    hostname: backrest
    volumes:
      - /srv/server/services/backrest/data:/data
      - /srv/server/services/backrest/config:/config
      - /srv/server/services/backrest/cache:/cache
      - /srv/server:/userdata # [optional] mount local paths to backup here.
      - /srv/server/backup/repos:/repos # [optional] mount repos if using local storage, not necessary for remotes e.g. B2, S3, etc.
    environment:
      - BACKREST_DATA=/data
      - BACKREST_CONFIG=/config/config.json
      - XDG_CACHE_HOME=/cache
      - TZ=Europe/Warsaw
    restart: unless-stopped
    ports:
      - 9898:9898
```