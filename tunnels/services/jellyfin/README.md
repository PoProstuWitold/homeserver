# Jellyfin
The Free Software Media System

Remember to change your [timezone](https://kevinnovak.github.io/Time-Zone-Picker/). Default is ``Europe/Warsaw``.

``docker-compose.yml``
```yaml
version: "3.8"

services:
  jellyfin:
    image: jellyfin/jellyfin
    container_name: jellyfin
    environment:
      - TZ=Europe/Warsaw
    volumes:
      - /home/docker/jellyfin/config:/config
      - /home/docker/jellyfin/media:/data
    ports:
      - 8096:8096
    restart: unless-stopped
```