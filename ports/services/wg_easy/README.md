# WireGuard VPN Easy
[WireGuard Easy](https://github.com/wg-easy/wg-easy) is the easiest way to run WireGuard VPN + Web-based Admin UI.

I recommend accessing following services through this VPN tunnel and **NOT** Caddy Reverse proxy:

- Portainer
- Cockpit
- WireGuard WebUI itself
- Minecraft RCON console
- NextCloud Master

...and generally access sensitive services through VPN.

``docker-compose.yml``
```yaml
services:
  wg_easy:
    image: ghcr.io/wg-easy/wg-easy:latest
    container_name: wg_easy
    ports:
      - "51820:51820/udp" # VPN
      - "51821:51821/tcp" # WebUI
    environment:
      - LANG=en
      - WG_HOST=YOUR_PUBLIC_STATIC_IP_OR_DDNS_ADDRESS
      - PASSWORD_HASH=password_hashed_with_bcrypt
      - PORT=51821 # WebUI TCP port
      - WG_PORT=51820 # VPN UDP port
      - WG_CONFIG_PORT=51820 # HomeAssistant Plugin
      - WG_DEFAULT_ADDRESS=10.8.0.x
      - WG_DEFAULT_DNS=1.1.1.1
      - WG_MTU=1420
      - WG_PERSISTENT_KEEPALIVE=25
      - WG_ALLOWED_IPS=192.168.2.0/24, 10.0.1.0/24 # replace 192.168.X.X part if nessesary
      # UI Elements
      - UI_TRAFFIC_STATS=true
      - UI_CHART_TYPE=1 # (0 Charts disabled, 1 # Line chart, 2 # Area chart, 3 # Bar chart)
      # Hooks
      # - WG_PRE_UP=echo "Pre Up" > /etc/wireguard/pre-up.txt
      # - WG_POST_UP=echo "Post Up" > /etc/wireguard/post-up.txt
      # - WG_PRE_DOWN=echo "Pre Down" > /etc/wireguard/pre-down.txt
      # - WG_POST_DOWN=echo "Post Down" > /etc/wireguard/post-down.txt
    volumes:
      - /srv/server/services/wg_easy:/etc/wireguard
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    sysctls:
      - net.ipv4.ip_forward=1
      - net.ipv4.conf.all.src_valid_mark=1
    restart: unless-stopped
```