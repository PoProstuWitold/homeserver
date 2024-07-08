# Caddy
[Caddy Reverse Proxy with Cloudflare DNS, Trusted Proxy and CrowdSec addons](https://github.com/serfriz/caddy-custom-builds/tree/main/caddy-cloudflare-ddns-crowdsec-geoip-security) is a powerful, enterprise-ready, open source web server with automatic HTTPS written in Go. This Docker image adds Cloudflare LetsEncrypt support to the base image.

Fill ``CLOUDFLARE_API_TOKEN`` according to docs from link above.

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
{
	debug

	log default {
		level debug
	}

	# caddy-cloudflare-ip
	servers {
		trusted_proxies cloudflare
		client_ip_headers CF-Connecting-IP
	}
}

(web) {
	encode zstd gzip

	# caddy-cloudflare-dns
	tls {
		dns cloudflare {env.CLOUDFLARE_API_TOKEN}
	}
}

*.{env.BASE_URL} {
	import web

	# Internet
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

	@linkwarden host linkwarden.{env.BASE_URL}
	handle @linkwarden {
		reverse_proxy linkwarden:3000
	}

	# Error handling
	@not_found host *.{env.BASE_URL}
	handle @not_found {
		respond "Not Found" 404
	}
}

{env.BASE_URL} {
	import web
	reverse_proxy homarr:7575
}
```


``docker-compose.yml``
```yaml
services:
  caddy:
    image: serfriz/caddy-cloudflare-ddns-crowdsec-geoip-security:latest
    container_name: caddy
    cap_add:
      - NET_ADMIN
    ports:
      - "80:80" # HTTP port
      - "443:443" # HTTPS port
      - "443:443/udp" # HTTP/3 port (optional)
    networks:
      - caddy
      - nextcloud-aio
    volumes:
      - /srv/server/services/caddy/data:/data
      - /srv/server/services/caddy/config:/config
      - /srv/server/services/caddy/Caddyfile:/etc/caddy/Caddyfile
      - /srv/server/services/caddy/logs:/var/log/caddy
    environment:
      - CLOUDFLARE_API_TOKEN=YOUR_CLOUDFLARE_API_TOKEN
      - ACME_AGREE=true
      - BASE_URL=your-domain.tld
    restart: unless-stopped

networks:
  caddy:
    name: caddy
    external: true
  nextcloud-aio:
    name: nextcloud-aio
    external: true
```