# WireGuard VPN Easy
[WireGuard Easy](https://github.com/wg-easy/wg-easy) is the easiest way to run WireGuard VPN + Web-based Admin UI.

I recommend accessing following services through this VPN tunnel and **NOT** Caddy Reverse proxy:

- Portainer
- Cockpit
- WireGuard WebUI itself
- Minecraft RCON console
- NextCloud Master

...and generally access sensitive services through VPN.

**If you encounter error ``Command failed`` or similar:**
1. Load ``ip6table_nat`` module into kernel
```bash
sudo modprobe ip6table_nat
```

2. Make sure it's installed
```bash
lsmod | grep ip6table_nat
```

3. Make this change permanent
```bash
echo ip6table_nat | sudo tee /etc/modules-load.d/wg-easy.conf
```

``docker-compose.yml``
```yaml
services:
  wg_easy:
    image: ghcr.io/wg-easy/wg-easy:15
    container_name: wg_easy
    environment:
      - PORT=51821
      - HOST=0.0.0.0
      - INSECURE=true
    volumes:
      - /srv/server/services/wg_easy:/etc/wireguard
      - /lib/modules:/lib/modules:ro
    ports:
      - "51820:51820/udp"
      - "51821:51821/tcp"
    networks:
      wg:
        ipv4_address: 10.42.42.42
        ipv6_address: fdcc:ad94:bacf:61a3::2a
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    sysctls:
      - net.ipv4.ip_forward=1
      - net.ipv4.conf.all.src_valid_mark=1
      - net.ipv6.conf.all.disable_ipv6=0
      - net.ipv6.conf.all.forwarding=1
      - net.ipv6.conf.default.forwarding=1
    restart: unless-stopped

networks:
  wg:
    driver: bridge
    enable_ipv6: true
    ipam:
      driver: default
      config:
        - subnet: 10.42.42.0/24
        - subnet: fdcc:ad94:bacf:61a3::/64
```