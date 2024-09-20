# Filebrowser
A Web File Browser.

``config/settings.json``
```json
{
	"port": 80,
	"baseURL": "",
	"address": "",
	"log": "stdout",
	"database": "/database/filebrowser.db",
	"root": "/srv"
}
```

``docker-compose.yml``
```yaml
services:
  filebrowser:
    image: filebrowser/filebrowser:s6
    container_name: filebrowser
    ports:
      - 8089:80
    environment:
      - PUID=0
      - PGID=0
    volumes:
      - /srv:/srv
      - /srv/server/services/filebrowser/database/filebrowser.db:/database/filebrowser.db
      - /srv/server/services/filebrowser/config/settings.json:/config/settings.json
    restart: unless-stopped
```