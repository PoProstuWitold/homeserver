# HomeAssistant
[HomeAssistant](https://github.com/home-assistant/core) is a source home automation tool that puts local control and privacy first. Powered by a worldwide community of tinkerers and DIY enthusiasts.

``docker-compose.yml``
```yaml
services:
  homeassistant:
    image: "ghcr.io/home-assistant/home-assistant:stable"
    container_name: homeassistant
    ports:
      - 8123:8123
    privileged: true
    volumes:
      - /srv/server/services/homeassistant/config:/config
      - /etc/localtime:/etc/localtime:ro
      - /run/dbus:/run/dbus:ro
    restart: unless-stopped
```