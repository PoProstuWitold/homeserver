# dash.
A modern server dashboard

demo: [dash.mauz.dev](https://dash.mauz.dev/)
docs: [getdashdot.com](https://getdashdot.com/)

``docker-compose.yml``
```yaml
version: "3.8"

services:
  dash:
    container_name: dash
    image: mauricenino/dashdot:latest
    restart: unless-stopped
    privileged: true
    ports:
      - '80:3001'
    volumes:
      - /:/mnt/host:ro
```