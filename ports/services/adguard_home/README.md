# AdGuard Home DNS Setup

This directory contains the DNS setup used in my homeserver environment.

The goal of this setup is to provide:

- local DNS resolution for LAN and VPN clients,
- split DNS for private `*.lan` subdomains,
- DNS-based ad/tracker filtering,
- safe access to private admin panels without exposing them publicly,
- compatibility with Caddy reverse proxy and Cloudflare DNS challenge certificates.

> This configuration is intended for a private homelab environment.  
> Replace all example domains, IP addresses and network ranges with your own values.

---

## Overview

The network uses:

- **AdGuard Home** as the local DNS resolver,
- **ASUS router** as the DHCP server,
- **WireGuard / wg-easy** for VPN access,
- **Caddy** as the reverse proxy,
- **Cloudflare** as public DNS provider and DNS challenge provider for TLS certificates.

Public services are served through normal public subdomains, for example:

```text
app.your-domain.tld
nextcloud.your-domain.tld
gitea.your-domain.tld
```

Private services are served through the internal `lan` subdomain:

```text
pufferpanel.lan.your-domain.tld
portainer.lan.your-domain.tld
filebrowser.lan.your-domain.tld
adguard.lan.your-domain.tld
```

The private `*.lan.your-domain.tld` records resolve to the local homeserver IP only when using AdGuard Home from LAN or VPN.

---

## Network Assumptions

Placeholder values used in this README:

```text
LAN subnet:          LAN_SUBNET
Router IP:           ROUTER_IP
Homeserver IP:       HOMESERVER_IP
WireGuard subnet:    VPN_SUBNET
Domain:              your-domain.tld
Private DNS zone:    lan.your-domain.tld
```

Replace these placeholders with values from your own network:

```text
LAN_SUBNET                   -> your local LAN subnet
ROUTER_IP                    -> your router IP address
HOMESERVER_IP                -> your homeserver IP address
VPN_SUBNET                   -> your WireGuard client subnet
CADDY_WG_NAT_IP              -> optional Docker/WireGuard NAT address seen by Caddy
OPTIONAL_IPV6_PRIVATE_SUBNET -> optional private IPv6 subnet, if used
```

In my setup, the router remains responsible for DHCP.  
AdGuard Home is used only as DNS.

---

## Docker Compose

`docker-compose.yml`:

```yaml
services:
  adguard_home:
    image: adguard/adguardhome:latest
    container_name: adguard_home
    hostname: adguard_home
    environment:
      - TZ=Europe/Warsaw
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE
    networks:
      - caddy_private
    ports:
      - "53:53/tcp"
      - "53:53/udp"
      - 3007:3000
    volumes:
      - /srv/server/services/adguard_home/work:/opt/adguardhome/work
      - /srv/server/services/adguard_home/conf:/opt/adguardhome/conf
    restart: unless-stopped

networks:
  caddy_private:
    external: true
```

The AdGuard Home setup/admin port `3007` should be reachable only from LAN/VPN and must not be port-forwarded to the public internet.

Create directories:

```bash
mkdir -p ./work ./conf
```

Start the service:

```bash
docker compose up -d
```

---

## Router DNS Settings

The ASUS router stays as the DHCP server.

Recommended DNS settings:

```text
DNS Server 1: HOMESERVER_IP
DNS Server 2: empty
Advertise router's IP in addition to user-specified DNS: Yes
```

This means clients should receive:

```text
DNS 1: HOMESERVER_IP  # AdGuard Home
DNS 2: ROUTER_IP   # router fallback
```

This is a practical balance between split DNS and resilience.

If the homeserver goes down, clients can still use the router as fallback DNS.

> Note: some clients may use the fallback DNS even when AdGuard Home is available.  
> For strict split DNS, use only AdGuard Home or deploy a second local DNS resolver.

---

## WireGuard / wg-easy DNS

For VPN clients, set DNS to AdGuard Home:

```ini
DNS = HOMESERVER_IP
```

The VPN peer must also have a route to the LAN subnet.

For split tunneling:

```ini
AllowedIPs = VPN_SUBNET, LAN_SUBNET
```

For full tunneling:

```ini
AllowedIPs = 0.0.0.0/0, ::/0
```

If the VPN client does not use AdGuard Home as DNS, private `*.lan.your-domain.tld` domains will resolve through public DNS instead of the local DNS rewrite.

---

## AdGuard Home DNS Rewrites

In AdGuard Home:

```text
Filters -> DNS rewrites
```

Add:

```text
lan.your-domain.tld       -> HOMESERVER_IP
*.lan.your-domain.tld     -> HOMESERVER_IP
```

Examples:

```text
pufferpanel.lan.your-domain.tld  -> HOMESERVER_IP
portainer.lan.your-domain.tld    -> HOMESERVER_IP
filebrowser.lan.your-domain.tld  -> HOMESERVER_IP
adguard.lan.your-domain.tld      -> HOMESERVER_IP
```

This allows private services to use clean HTTPS URLs while staying available only from LAN/VPN.

---

## Cloudflare DNS Records

If using a public wildcard record like:

```text
*.your-domain.tld    Proxied
your-domain.tld      Proxied
```

Cloudflare may also answer public DNS queries for deeper subdomains such as:

```text
pufferpanel.lan.your-domain.tld
```

To prevent private `*.lan` names from being proxied by the public wildcard, add explicit DNS-only placeholder records:

```text
lan.your-domain.tld       A    192.0.2.1    DNS only
*.lan.your-domain.tld     A    192.0.2.1    DNS only
```

Why `192.0.2.1`?

`192.0.2.0/24` is reserved for documentation and examples. It is used here as a harmless placeholder so public DNS does not point private names to the real server or Cloudflare proxy.

Local AdGuard Home rewrites override these records inside LAN/VPN.

Final behavior:

```text
LAN/VPN DNS via AdGuard:
pufferpanel.lan.your-domain.tld -> HOMESERVER_IP

Public DNS:
pufferpanel.lan.your-domain.tld -> 192.0.2.1
```

---

## Caddy Integration

Caddy is used as the HTTPS reverse proxy for both public and private services.

The private `*.lan.your-domain.tld` services should only be reachable from trusted LAN/VPN networks.

Example shared snippets:

```caddyfile
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
        @private client_ip YOUR_LAN_SUBNET YOUR_VPN_SUBNET CADDY_WG_NAT_IP/32 OPTIONAL_IPV6_PRIVATE_SUBNET

        handle @private {
                import secure *
                reverse_proxy {args[0]}
        }

        respond "Forbidden" 403
}
```

Replace:

```txt
YOUR_LAN_SUBNET               -> your LAN subnet, for example 192.168.x.0/24
YOUR_VPN_SUBNET               -> your WireGuard subnet, for example 10.x.x.0/24
CADDY_WG_NAT_IP/32            -> optional Docker/WireGuard NAT IP seen by Caddy
OPTIONAL_IPV6_PRIVATE_SUBNET  -> optional private IPv6 subnet, remove if unused
```

Example private LAN/VPN test endpoint:

```caddyfile
lan.{env.BASE_URL} {
        import web

        @private client_ip YOUR_LAN_SUBNET YOUR_VPN_SUBNET CADDY_WG_NAT_IP/32 OPTIONAL_IPV6_PRIVATE_SUBNET

        handle @private {
                respond "Private LAN/VPN zone - access granted" 200
        }

        respond "Forbidden" 403
}
```

Example private wildcard block:

```caddyfile
*.lan.{env.BASE_URL} {
        import web

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

        @metrics host metrics.lan.{env.BASE_URL}
        handle @metrics {
                @private client_ip YOUR_LAN_SUBNET YOUR_VPN_SUBNET CADDY_WG_NAT_IP/32 OPTIONAL_IPV6_PRIVATE_SUBNET

                handle @private {
                        metrics
                }

                respond "Forbidden" 403
        }

        @homeassistant host home.lan.{env.BASE_URL}
        handle @homeassistant {
                @private client_ip YOUR_LAN_SUBNET YOUR_VPN_SUBNET CADDY_WG_NAT_IP/32 OPTIONAL_IPV6_PRIVATE_SUBNET

                handle @private {
                        reverse_proxy homeassistant:8123
                }

                respond "Forbidden" 403
        }

        handle {
                respond "Not Found" 404
        }
}
```

Example private domains:

```txt
lan.your-domain.tld
adguard.lan.your-domain.tld
filebrowser.lan.your-domain.tld
pufferpanel.lan.your-domain.tld
grafana.lan.your-domain.tld
uptime.lan.your-domain.tld
home.lan.your-domain.tld
```

Do not expose through public Caddy routes:

- AdGuard Home admin panel
- Portainer
- FileBrowser
- wg-easy WebUI
- Zigbee2MQTT frontend
- Mosquitto MQTT
- SLZB-06MU panel
- internal dashboards

---

## Caddy Validation

Validate and reload Caddy:

```bash
docker exec -it caddy caddy validate --config /etc/caddy/Caddyfile
docker exec -it caddy caddy reload --config /etc/caddy/Caddyfile
```

Check logs:

```bash
docker logs caddy --tail=100
```

---

## Testing DNS

From LAN or VPN:

```bash
resolvectl query lan.your-domain.tld
resolvectl query pufferpanel.lan.your-domain.tld
```

Expected result:

```text
HOMESERVER_IP
```

Force query directly to AdGuard Home:

```bash
nslookup lan.your-domain.tld HOMESERVER_IP
nslookup pufferpanel.lan.your-domain.tld HOMESERVER_IP
```

Expected result:

```text
Address: HOMESERVER_IP
```

From outside LAN/VPN, public DNS should return the placeholder:

```text
192.0.2.1
```

---

## Testing Access

From LAN/VPN:

```bash
curl -I https://pufferpanel.lan.your-domain.tld
```

Expected result:

```text
HTTP/2 200
```

or a redirect to Authelia.

From outside LAN/VPN:

```bash
curl -I https://pufferpanel.lan.your-domain.tld
```

Expected result:

```text
Connection failure
```

or:

```text
HTTP/2 403
```

depending on DNS and routing.

---

## Troubleshooting

### DNS returns Cloudflare IP instead of local IP

Problem:

```text
lan.your-domain.tld -> 104.x.x.x / 172.67.x.x / 2606:4700:...
```

Possible causes:

- client is not using AdGuard Home,
- DNS cache still contains old Cloudflare response,
- VPN client has wrong DNS,
- Android Private DNS / browser Secure DNS bypasses local DNS.

Fix:

```bash
resolvectl flush-caches
```

Check resolver:

```bash
resolvectl status
resolvectl query lan.your-domain.tld
```

For WireGuard clients, verify:

```ini
DNS = HOMESERVER_IP
```

### `nslookup` uses the wrong DNS server

If `resolvectl query` works but `nslookup` uses a different DNS, check `/etc/resolv.conf`:

```bash
ls -l /etc/resolv.conf
cat /etc/resolv.conf
```

If systemd-resolved is used, it should usually point to:

```text
/run/systemd/resolve/stub-resolv.conf
```

Fix:

```bash
sudo rm /etc/resolv.conf
sudo ln -s /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
sudo systemctl restart systemd-resolved
resolvectl flush-caches
```

### Caddy returns `403` over VPN

Check what IP Caddy sees:

```caddyfile
lan.{env.BASE_URL} {
        import web
        respond "client_ip={client_ip} remote_host={http.request.remote.host} xff={http.request.header.X-Forwarded-For} cf={http.request.header.CF-Connecting-IP} host={http.request.host}" 200
}
```

If Caddy sees traffic from a Docker/WireGuard NAT address, add that address to the private allowlist.

Example:

```text
client_ip=CADDY_WG_NAT_IP
```

Then add:

```caddyfile
CADDY_WG_NAT_IP/32
```

to the `@private` matcher.

### TLS errors for `*.lan.your-domain.tld`

A certificate for:

```text
*.your-domain.tld
```

does not cover:

```text
pufferpanel.lan.your-domain.tld
```

For private service subdomains, Caddy needs a certificate for:

```text
*.lan.your-domain.tld
```

Using DNS challenge with Cloudflare allows Caddy to issue this certificate without exposing private services publicly.

---

## Security Notes

- Do not expose AdGuard Home admin panel directly to the internet.
- Do not port-forward DNS port `53` from WAN to the homeserver.
- Do not rely only on DNS for access control.
- Private services should also be protected in Caddy using LAN/VPN allowlists.
- Sensitive admin panels should additionally use SSO/2FA, for example Authelia.
- Public Cloudflare records for `*.lan.your-domain.tld` should not be proxied.
- Use DNS-only placeholder records for `lan` and `*.lan` if a public wildcard exists.

---

## Backup

Backup these directories:

```text
./conf
./work
```

The most important file is:

```text
./conf/AdGuardHome.yaml
```

It contains AdGuard Home settings, DNS rewrites, upstream DNS configuration and filtering configuration.

Do not commit real production configuration with private IP addresses, upstream DNS credentials, API tokens or other secrets to a public repository.

---

## Summary

This setup provides:

- local DNS for LAN and VPN clients,
- private HTTPS subdomains under `*.lan.your-domain.tld`,
- public DNS isolation using Cloudflare DNS-only placeholder records,
- valid TLS certificates through DNS challenge,
- private access enforcement in Caddy,
- router DNS fallback for better home network resilience.

It is designed for a self-hosted homelab where private admin panels should have nice HTTPS URLs without being exposed to the public internet.
