# Caddy
[Caddy Reverse Proxy with Cloudflare DNS addon](https://github.com/SlothCroissant/caddy-cloudflaredns) is a powerful, enterprise-ready, open source web server with automatic HTTPS written in Go. This Docker image adds Cloudflare LetsEncrypt support to the base image.

Fill ``CLOUDFLARE_EMAIL`` and ``CLOUDFLARE_API_TOKEN`` according to docs from link above.

Before using image you have to create Docker network named ``caddy`` either with Portainer or command:

```bash
docker network create caddy
```

It will be **really** important later on because if both *caddy* and some other web service (e.g. *homarr*, *mealie*) are in the same network, they can talk to each other with their names which makes managing server easier.

Additionaly you can do same with network named ``nextcloud-aio`` if you want to use **NextCloud AIO** image. The reason for this is that *NextCloud AIO* spawns many containers and automatically adds them to network ``nextcloud-aio`` with no possibility to add them to different network.

Example config file for Caddy:

``Caddyfile``
```python
*.your-domain.tld {
  @mealie host mealie.your-domain.tld
  handle @mealie {
    # note that the port must be the one inside Docker, not mapped one
    reverse_proxy mealie:9000
  }

  tls {
    dns cloudflare {env.CLOUDFLARE_API_TOKEN}
  }
}

your-domain.tld {
  reverse_proxy homarr:7575

  tls {
    dns cloudflare {env.CLOUDFLARE_API_TOKEN}
  }
}
```


``docker-compose.yml``
```yaml
services:
  caddy:
    image: slothcroissant/caddy-cloudflaredns:latest
    container_name: caddy
    labels:
    # makes watchtower not update caddy but still monitor it
      - "com.centurylinklabs.watchtower.monitor-only=true"
    ports:
      - 80:80
      - 443:443
    networks:
      - caddy
      # uncomment if you want to use nextcloud-aio
      # - nextcloud-aio
    volumes:
      - /srv/server/services/caddy/data:/data
      - /srv/server/services/caddy/config:/config
      - /srv/server/services/caddy/Caddyfile:/etc/caddy/Caddyfile
    environment:
      - CLOUDFLARE_EMAIL=YOUR_CLOUDFLARE_EMAIL
      - CLOUDFLARE_API_TOKEN=YOUR_CLOUDFLARE_API_TOKEN
      - ACME_AGREE=true
    restart: unless-stopped

networks:
  caddy:
    name: caddy
    external: true
  # uncomment if you want to use nextcloud-aio
  # nextcloud-aio:
    # name: nextcloud-aio
    # external: true
```