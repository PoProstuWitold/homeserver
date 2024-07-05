# Portainer
Powerful container management

``docker-compose.yml``
```yaml
services:
  portainer:
    container_name: portainer
    image: portainer/portainer-ce:latest
    ports:
      - 9000:9000
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /srv/server/services/portainer/data:/data
    restart: unless-stopped
```