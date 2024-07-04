# Jellyfin
The Free Software Media System

Remember to change your [timezone](https://kevinnovak.github.io/Time-Zone-Picker/). Default is ``Europe/Warsaw``.

``docker-compose.yml``
```yaml
services:
  jellyfin:
    image: jellyfin/jellyfin:latest
    container_name: jellyfin
    ports:
      - 8096:8096
    environment:
      - TZ=Europe/Warsaw
    volumes:
      - /srv/server/services/jellyfin/config:/config
      - /srv/server/media:/data
    restart: unless-stopped
```