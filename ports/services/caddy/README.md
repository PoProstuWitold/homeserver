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
# global acme_dns option wasn't working for me for some reason
(tls) {
  tls {
    dns cloudflare {env.CLOUDFLARE_API_TOKEN}
  }
}

*.{env.BASE_URL} {
  import tls

  @mealie host mealie.{env.BASE_URL}
  handle @mealie {
    reverse_proxy mealie:9000
  }

  @dashdot host dashdot.{env.BASE_URL}
  handle @dashdot {
    reverse_proxy dashdot:3001
  }

  @nextcloud host nextcloud.{env.BASE_URL}
  handle @nextcloud {
    reverse_proxy nextcloud-aio-apache:11000
  }

  # handle not found error, because Caddy
  # by default returns status 200 and empty page
  @not_found host *.{env.BASE_URL}
  handle @not_found {
    respond "Not Found" 404
  }
}

{env.BASE_URL} {
  import tls
  reverse_proxy homarr:7575
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
      - BASE_URL=your-domain.tld
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