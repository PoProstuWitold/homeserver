# Omni Tools
[Omni Tools](https://github.com/iib0011/omni-tools) is a self-hosted collection of powerful web-based tools for everyday tasks. No ads, no tracking, just fast, accessible utilities right from your browser! 

``docker-compose.yml``
```yaml
services:
  omni_tools:
    image: iib0011/omni-tools:latest
    container_name: omni_tools
    ports:
      - "8082:80"
    networks:
      - caddy
    restart: unless-stopped

networks:
  caddy:
    name: caddy
    external: true
```