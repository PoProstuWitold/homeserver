# Jellyseerr
Application For Managing Requests For Your Media Library. Requires *Jellyfin* to be installed first.

I also recommend to install these services first:
- Prowlarr
- Radarr
- Sonarr

all of them are docummented in ***services/star_apps*** directory.

Remember to change your [timezone](https://kevinnovak.github.io/Time-Zone-Picker/). Default is ``Europe/Warsaw``.

``docker-compose.yml``
```yaml
services:
  jellyseerr:
    image: fallenbagel/jellyseerr:latest
    container_name: jellyseerr
    ports:
      - 5055:5055
    environment:
      - LOG_LEVEL=debug
      - TZ=Europe/Warsaw
    volumes:
      - /srv/server/services/jellyseer/config:/app/config
    restart: unless-stopped
```