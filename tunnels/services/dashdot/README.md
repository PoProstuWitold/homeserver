# dashdot
A modern server dashboard

demo: [dash.mauz.dev](https://dash.mauz.dev/)
docs: [getdashdot.com](https://getdashdot.com/)

``docker-compose.yml``
```yaml
services:
  dashdot:
    container_name: dashdot
    image: mauricenino/dashdot:latest
    restart: unless-stopped
    privileged: true
    ports:
      - 3001:3001
    volumes:
      - /:/mnt/host:ro
```