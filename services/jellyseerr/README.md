# Jellyseerr
Application For Managing Requests For Your Media Library. Requires *Jellyfin* to be installed first.

I also recommend to install these services first:
- Prowlarr
- Radarr
- Sonarr
all of them are docummented in ***star_apps*** directory.

Remember to change your [timezone](https://kevinnovak.github.io/Time-Zone-Picker/). Default is ``Europe/Warsaw``.

``docker-compose.yml``
```yaml
version: "3.8"

services:
  jellyseerr:
    image: fallenbagel/jellyseerr:latest
    container_name: jellyseerr
    environment:
      - LOG_LEVEL=debug
      - TZ=Europe/Warsaw
    ports:
      - 5055:5055
    volumes:
      - /home/docker/jellyseer/config:/app/config
    restart: unless-stopped
```