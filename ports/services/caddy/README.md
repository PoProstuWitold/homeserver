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
{
    debug
    log default {
        format json
        output file /var/log/caddy/access.log {
            roll_size 100MiB
            roll_keep 10
            roll_keep_for 100d
        }
    }

    # crowdsec
    crowdsec {
        api_url http://crowdsec:8080
        api_key {env.CROWDSEC_API_KEY}
    }
    order crowdsec first

    # caddy-cloudflare-ip
    servers {
        trusted_proxies cloudflare
        client_ip_headers CF-Connecting-IP
        metrics
    }
}

(web) {
    crowdsec
    encode zstd gzip

    handle_errors {
        rewrite * /{http.error.status_code}
        reverse_proxy https://http.cat
    }

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

    @uptime_kuma host uptime.{env.BASE_URL}
    handle @uptime_kuma {
        reverse_proxy uptime_kuma:3001
    }

    @grafana host grafana.{env.BASE_URL}
    handle @grafana {
        reverse_proxy grafana:3000
    }

    @jellyfin host jellyfin.{env.BASE_URL}
    handle @jellyfin {
        reverse_proxy jellyfin:8096
    }

    @jellyseerr host jellyseerr.{env.BASE_URL}
    handle @jellyseerr {
        reverse_proxy jellyseerr:5055
    }

    # Metrics
    @metrics host metrics.{env.BASE_URL}
    handle @metrics {
        metrics
    }

    # Not Found
    @not_found host *.{env.BASE_URL}
    handle @not_found {
        respond "Not Found" 404
    }
}

{env.BASE_URL} {
    import web

    @geoip_filter {
        maxmind_geolocation {
            db_path "/etc/caddy/GeoLite2-City.mmdb"
            allow_countries PL
        }
    }

    reverse_proxy @geoip_filter homarr:7575
}
```

## Crowdsec

If you want to use **Crowdsec** module, you need to do a couple of things.

1. Create ``acquis.yaml`` file in your chosen location and fill it with following:

```yaml
filenames:
  - /var/log/caddy/*.log
labels:
  type: caddy
```

2. Generate ``CROWDSEC_API_KEY``

You can do this with command:

```bash
echo "CROWDSEC_API_KEY=$(tr -dc A-Za-z0-9 </dev/urandom | head -c 32)"
```

After that paste it to ``CROWDSEC_API_KEY`` in **caddy** service and ``BOUNCER_KEY_CADDY`` in **crowdsec** service.

## GeoLite2

If you want to use **GeoLite2** module, you need to do a couple of things.

1. Create developer account on [dev.maxmind.com](https://dev.maxmind.com/geoip/geolite2-free-geolocation-data).

2. Get your ``ACCOUNT_ID`` and ``LICENSE_KEY``. Instructions how to do that are at [dev.maxmind.com](https://dev.maxmind.com/geoip/updating-databases#directly-downloading-databases)

3. Head to your **caddy** directory and run command:

```bash
sudo wget --content-disposition --user=ACCOUNT_ID --password=LICENSE_KEY 'https://download.maxmind.com/geoip/databases/GeoLite2-City/download?suffix=tar.gz'
```

4. Untar your map. Note that your downloaded map name may be different:

```bash
sudo tar -xvzf GeoLite2-City_20240709.tar.gz 
```

5. Move your ``GeoLite2-City.mmdb`` file to root **caddy** directory:

```bash
sudo mv ./GeoLite2-City_20240709/GeoLite2-City.mmdb ./GeoLite2-City.mmdb 
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
      - /srv/server/services/caddy/GeoLite2-City.mmdb:/etc/caddy/GeoLite2-City.mmdb
    environment:
      - CLOUDFLARE_API_TOKEN=YOUR_CLOUDFLARE_API_TOKEN
      - ACME_AGREE=true
      - BASE_URL=your-domain.tld
      - CROWDSEC_API_KEY=
    restart: unless-stopped
    
  crowdsec:
    image: docker.io/crowdsecurity/crowdsec:latest
    container_name: crowdsec
    environment:
      - GID=1000
      - COLLECTIONS=crowdsecurity/caddy crowdsecurity/http-cve crowdsecurity/whitelist-good-actors
      - BOUNCER_KEY_CADDY=
    volumes:
      - /srv/server/services/crowdsec/db:/var/lib/crowdsec/data/
      - /srv/server/services/crowdsec/acquis.yaml:/etc/crowdsec/acquis.yaml
      - /srv/server/services/caddy/logs:/var/log/caddy:ro
    ports:
      - 6060:6060 # Prometheus API endpoint
    networks:
      - caddy
    restart: unless-stopped
    security_opt:
      - no-new-privileges=true

networks:
  caddy:
    name: caddy
    external: true
  nextcloud-aio:
    name: nextcloud-aio
    external: true
```