# HomeAssistant
[HomeAssistant](https://github.com/home-assistant/core) is a source home automation tool that puts local control and privacy first. Powered by a worldwide community of tinkerers and DIY enthusiasts.

``docker-compose.yml``
```yaml
version: "3.8"

services:
  homeassistant:
    container_name: homeassistant
    image: "ghcr.io/home-assistant/home-assistant:stable"
    volumes:
      - /home/docker/homeassistant/config:/config
      - /etc/localtime:/etc/localtime:ro
      - /run/dbus:/run/dbus:ro
    ports:
      - "8123:8123"
    restart: unless-stopped
    privileged: true
```