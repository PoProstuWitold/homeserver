# Caddy
[Caddy Reverse Proxy with Cloudflare DNS, Trusted Proxy and CrowdSec addons](https://github.com/serfriz/caddy-custom-builds/tree/main/caddy-cloudflare-ddns-crowdsec-geoip-security) is a powerful, enterprise-ready, open source web server with automatic HTTPS written in Go. This Docker image adds Cloudflare Let's Encrypt support to the base image.

Set ``CLOUDFLARE_API_TOKEN`` according to the documentation linked above.

Before using this image, create a Docker network named ``caddy`` either with Portainer or command:

```bash
docker network create caddy
```

## DNS records

This Caddy setup assumes that public DNS is handled by **Cloudflare**, while private LAN/VPN DNS records are handled by **AdGuard Home**.

More details about split DNS, private `*.lan` subdomains and DNS rewrites are described in the **AdGuard Home** service documentation in this repository.

### Cloudflare DNS records

| Type | Name | Content | Proxy status | Purpose |
|---|---|---|---|---|
| `A` | `your-domain.tld` | `PUBLIC_SERVER_IP` | Proxied | Public root domain |
| `A` | `*.your-domain.tld` | `PUBLIC_SERVER_IP` | Proxied | Public wildcard services |
| `A` | `lan.your-domain.tld` | `192.0.2.1` | DNS only | Placeholder for private LAN zone |
| `A` | `*.lan.your-domain.tld` | `192.0.2.1` | DNS only | Placeholder for private LAN wildcard |

`192.0.2.1` is a documentation/example IP address. It is used here only as a harmless public placeholder for private `*.lan` records.

Real private LAN/VPN resolution is handled by AdGuard Home.

### AdGuard Home DNS rewrites

| Domain | Target | Purpose |
|---|---|---|
| `lan.your-domain.tld` | `HOMESERVER_IP` | Private LAN/VPN test endpoint |
| `*.lan.your-domain.tld` | `HOMESERVER_IP` | Private LAN/VPN wildcard services |

Example private records resolved by AdGuard Home:

| Domain | Target |
|---|---|
| `adguard.lan.your-domain.tld` | `HOMESERVER_IP` |
| `filebrowser.lan.your-domain.tld` | `HOMESERVER_IP` |
| `pufferpanel.lan.your-domain.tld` | `HOMESERVER_IP` |
| `grafana.lan.your-domain.tld` | `HOMESERVER_IP` |
| `uptime.lan.your-domain.tld` | `HOMESERVER_IP` |
| `home.lan.your-domain.tld` | `HOMESERVER_IP` |

Replace:

- `your-domain.tld` with your real domain,
- `PUBLIC_SERVER_IP` with your public server IP address,
- `HOMESERVER_IP` with your local homeserver IP address.

Do not point public `*.lan` records to the real homeserver IP address.


It will be **really** important later on because if both *caddy* and some other web service (e.g. *homarr*, *mealie*) are in the same network, they can talk to each other with their names which makes managing server easier.

Additionally you can do same with network named ``nextcloud-aio`` if you want to use **NextCloud AIO** image. The reason for this is that *NextCloud AIO* spawns many containers and automatically adds them to network ``nextcloud-aio`` with no possibility to add them to different network.

Example config file for Caddy:

``Caddyfile``
```ini
{
        debug

        log default {
                format json
                output file /var/log/caddy/caddy.log {
                        mode 0644
                        roll_size 100MiB
                        roll_keep 10
                        roll_keep_for 100d
                }
                exclude http.log.access
        }

        log access {
                format json
                output file /var/log/caddy/caddy-access.log {
                        mode 0644
                        roll_size 100MiB
                        roll_keep 10
                        roll_keep_for 14d
                }
                include http.log.access
        }

        # CrowdSec bouncer integration
        crowdsec {
                api_url http://crowdsec:8080
                api_key {env.CROWDSEC_API_KEY}
        }

        order crowdsec first

        # Trust Cloudflare only as reverse proxy
        servers {
                trusted_proxies cloudflare
                client_ip_headers CF-Connecting-IP
        }
}

(web) {
        crowdsec
        encode zstd gzip

        log

        handle_errors {
                rewrite * /{http.error.status_code}
                reverse_proxy https://http.cat
        }

        tls {
                dns cloudflare {env.CLOUDFLARE_API_TOKEN}
        }
}

(secure) {
        forward_auth {args[0]} authelia:9091 {
                uri /api/verify?rd=https://auth.{env.BASE_URL}
                copy_headers Remote-User Remote-Groups Remote-Name Remote-Email
        }
}

(private_secure) {
        # LAN, WireGuard clients and optional wg-easy/Docker NAT address seen by Caddy
        @private client_ip YOUR_LAN_SUBNET YOUR_VPN_SUBNET CADDY_WG_NAT_IP/32 OPTIONAL_IPV6_PRIVATE_SUBNET

        handle @private {
                import secure *
                reverse_proxy {args[0]}
        }

        respond "Forbidden" 403
}

# ============================================================
# Public wildcard services
# ============================================================

*.{env.BASE_URL} {
        import web

        @geoip {
                maxmind_geolocation {
                        db_path "/etc/caddy/GeoLite2-City.mmdb"
                        allow_countries YOUR_COUNTRY_CODE
                }
        }

        # Authentication
        @authelia host auth.{env.BASE_URL}
        handle @authelia {
                reverse_proxy @geoip authelia:9091
        }

        # My Apps
        @doggopaste host doggopaste.{env.BASE_URL}
        handle @doggopaste {
                import secure /dashboard
                reverse_proxy @geoip doggopaste:3002
        }

        @pizzeria host pizzeria.{env.BASE_URL}
        handle @pizzeria {
                reverse_proxy @geoip pizzeria:3005
        }

        @nuntius_feed host feed.{env.BASE_URL}
        handle @nuntius_feed {
                import secure *
                reverse_proxy @geoip nuntius_feed:3006
        }

        # Public Selfhosted Apps
        @nextcloud host nextcloud.{env.BASE_URL}
        handle @nextcloud {
                reverse_proxy @geoip nextcloud-aio-apache:11000
        }

        @mealie host mealie.{env.BASE_URL}
        handle @mealie {
                import secure *
                reverse_proxy @geoip mealie:9000
        }

        @linkwarden host linkwarden.{env.BASE_URL}
        handle @linkwarden {
                import secure *
                reverse_proxy @geoip linkwarden:3000
        }

        @jellyfin host jellyfin.{env.BASE_URL}
        handle @jellyfin {
                reverse_proxy @geoip jellyfin:8096
        }

        @jellyseerr host jellyseerr.{env.BASE_URL}
        handle @jellyseerr {
                import secure *
                reverse_proxy @geoip jellyseerr:5055
        }

        @gitea host gitea.{env.BASE_URL}
        handle @gitea {
                reverse_proxy @geoip gitea:3000
        }

        @sftpgo host sftp.{env.BASE_URL}
        handle @sftpgo {
                reverse_proxy @geoip sftpgo:8080
        }

        @omni_tools host tools.{env.BASE_URL}
        handle @omni_tools {
                reverse_proxy @geoip omni_tools:80
        }

        # Not Found
        handle {
                respond "Not Found" 404
        }
}

# ============================================================
# Public root domain
# ============================================================

{env.BASE_URL} {
        import web

        @geoip {
                maxmind_geolocation {
                        db_path "/etc/caddy/GeoLite2-City.mmdb"
                        allow_countries YOUR_COUNTRY_CODE
                }
        }

        handle @geoip {
                reverse_proxy homarr:7575
        }

        respond "Forbidden" 403
}

# ============================================================
# Private LAN/VPN dashboard
# ============================================================

lan.{env.BASE_URL} {
        import web

        @private client_ip YOUR_LAN_SUBNET YOUR_VPN_SUBNET CADDY_WG_NAT_IP/32 OPTIONAL_IPV6_PRIVATE_SUBNET

        handle @private {
                respond "LAN is working. GLHF!" 200
        }

        respond "Forbidden" 403
}

# ============================================================
# Private LAN/VPN services
# ============================================================

*.lan.{env.BASE_URL} {
        import web

        # Admin
        @adguard host adguard.lan.{env.BASE_URL}
        handle @adguard {
                import private_secure adguard_home:3000
        }

        @filebrowser host filebrowser.lan.{env.BASE_URL}
        handle @filebrowser {
                import private_secure filebrowser:80
        }

        @pufferpanel host pufferpanel.lan.{env.BASE_URL}
        handle @pufferpanel {
                import private_secure pufferpanel:8080
        }

        # Monitoring
        @grafana host grafana.lan.{env.BASE_URL}
        handle @grafana {
                import private_secure grafana:3000
        }

        @uptime_kuma host uptime.lan.{env.BASE_URL}
        handle @uptime_kuma {
                import private_secure uptime_kuma:3001
        }

        @dashdot host dashdot.lan.{env.BASE_URL}
        handle @dashdot {
                import private_secure dashdot:3001
        }

        # Caddy Metrics
        @metrics host metrics.lan.{env.BASE_URL}
        handle @metrics {
                @private client_ip YOUR_LAN_SUBNET YOUR_VPN_SUBNET CADDY_WG_NAT_IP/32 OPTIONAL_IPV6_PRIVATE_SUBNET

                handle @private {
                        metrics
                }

                respond "Forbidden" 403
        }

        # Misc/My Apps
        # Home Assistant
        @homeassistant host home.lan.{env.BASE_URL}
        handle @homeassistant {
                @private client_ip YOUR_LAN_SUBNET YOUR_VPN_SUBNET CADDY_WG_NAT_IP/32 OPTIONAL_IPV6_PRIVATE_SUBNET

                handle @private {
                        reverse_proxy homeassistant:8123
                }

                respond "Forbidden" 403
        }

        # Not Found
        handle {
                respond "Not Found" 404
        }
}
```

Replace:
- `YOUR_LAN_SUBNET` with your LAN subnet
- `YOUR_VPN_SUBNET` with your WireGuard subnet
- `CADDY_WG_NAT_IP/32` with the optional Docker/WireGuard NAT address seen by Caddy
- `OPTIONAL_IPV6_PRIVATE_SUBNET` with your private IPv6 subnet, or remove it if unused
- `YOUR_COUNTRY_CODE` with your GeoIP country code, or remove the GeoIP matcher if unused

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

3. Create custom whitelist

You are very likely to get banned yourself while toying with your homelab so I recommend to create a custom whitelist:
``/srv/server/services/crowdsec/config/parsers/s02-enrich/custom_whitelist.yaml``
```bash
name: homeserver/whitelists
description: "Whitelist of known, friendly IP addresses"
whitelist:
  reason: "Known addresses"
  ip:
    - "YOUR_SERVER_PUBLIC_IP"
    - "ANOTHER_TRUSTED_PUBLIC_IP"
```

Restart container and after executing command ``docker exec -it crowdsec cscli parsers list`` you can see that another parser - ``homeserver/whitelists`` - has been added.

## GeoLite2

If you want to use **GeoLite2** module, you need to do a couple of things.

In the example Caddyfile, replace `YOUR_COUNTRY_CODE` with the country code you want to allow, or remove the GeoIP matcher if you do not want country-based filtering.

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
      - caddy_private
      - nextcloud-aio
    volumes:
      - /srv/server/services/caddy/data:/data
      - /srv/server/services/caddy/config:/config
      - /srv/server/services/caddy/Caddyfile:/etc/caddy/Caddyfile
      - /srv/server/services/caddy/logs:/var/log/caddy
      - /srv/server/services/caddy/GeoLite2-City.mmdb:/etc/caddy/GeoLite2-City.mmdb
    environment:
      - CLOUDFLARE_API_TOKEN=
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
      - /srv/server/services/crowdsec/config:/etc/crowdsec/
      - /srv/server/services/caddy/logs:/var/log/caddy:ro
    ports:
      - 6060:6060 # Prometheus API endpoint
    networks:
      - caddy
      - caddy_private
    restart: unless-stopped
    security_opt:
      - no-new-privileges=true

networks:
  caddy:
    name: caddy
    external: true
  caddy_private:
    name: caddy_private
    external: true
  nextcloud-aio:
    name: nextcloud-aio
    external: true
```