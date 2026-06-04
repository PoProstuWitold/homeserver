# Home Assistant + Zigbee2MQTT + Mosquitto + SLZB-06MU

Secure local smart home stack for Zigbee devices.

Stack:
- Home Assistant
- Zigbee2MQTT
- Mosquitto MQTT
- SMLIGHT SLZB-06MU
- Docker Compose
- Caddy reverse proxy
- LAN/VPN-only access model

This setup keeps Zigbee, MQTT and coordinator management private. Home Assistant is exposed only through a controlled LAN/VPN route.

---

## Architecture

```txt
Zigbee devices
    ↓
SMLIGHT SLZB-06MU
    ↓ Ethernet / TCP
Zigbee2MQTT
    ↓ MQTT
Mosquitto
    ↓ MQTT integration
Home Assistant
    ↓ Caddy HTTPS reverse proxy
LAN/VPN clients
```

---

## Security model

Public internet should not reach:
- Mosquitto MQTT
- Zigbee2MQTT frontend
- SLZB-06MU web panel
- internal Docker networks
- MQTT credentials
- device administration panels

The setup avoids:
- `privileged: true`
- `network_mode: host`
- public MQTT port
- public Zigbee2MQTT frontend
- public SLZB-06MU panel

Home Assistant should be available through a private LAN/VPN domain, for example:

```txt
https://ha.lan.your-domain.tld
```

---

## Directory structure

```txt
/srv/server/services/smarthome
├── homeassistant
├── mosquitto
│   ├── config
│   ├── data
│   └── log
└── zigbee2mqtt
    └── data
```

Create directories:
```bash
sudo mkdir -p /srv/server/services/smarthome
cd /srv/server/services/smarthome

sudo mkdir -p homeassistant
sudo mkdir -p mosquitto/config mosquitto/data mosquitto/log
sudo mkdir -p zigbee2mqtt/data
```

---

## Mosquitto MQTT

Create config:
```bash
sudo nano /srv/server/services/smarthome/mosquitto/config/mosquitto.conf
```

`mosquitto.conf`:
```txt
persistence true
persistence_location /mosquitto/data/

listener 1883
allow_anonymous false
password_file /mosquitto/config/passwordfile
```

Create MQTT user:
```bash
sudo docker run --rm -it \
  -v /srv/server/services/smarthome/mosquitto/config:/mosquitto/config \
  eclipse-mosquitto:latest \
  mosquitto_passwd -c /mosquitto/config/passwordfile homeassistant
```

Fix permissions:
```bash
sudo docker run --rm -u root \
  -v /srv/server/services/smarthome/mosquitto/config:/mosquitto/config \
  eclipse-mosquitto:latest \
  sh -c "chown mosquitto:mosquitto /mosquitto/config/passwordfile && chmod 600 /mosquitto/config/passwordfile"
```

---

## Zigbee2MQTT

Create config:
```bash
sudo nano /srv/server/services/smarthome/zigbee2mqtt/data/configuration.yaml
```

`configuration.yaml`:
```yaml
homeassistant:
  enabled: true
frontend:
  enabled: true
  port: 8080
mqtt:
  server: mqtt://mosquitto:1883
  user: homeassistant
  password: MQTT_PASSWORD
serial:
  port: tcp://SLZB_06MU_IP:6638
  baudrate: 115200
  adapter: ember
  disable_led: false
advanced:
  channel: 15
  transmit_power: 20
```

Replace:
- `SLZB_06MU_IP` with the real coordinator IP
- `MQTT_PASSWORD` with the Mosquitto user password

SLZB-06MU connection:
- Ethernet to router/switch
- USB-C to power adapter

---

## Docker Compose

`docker-compose.yml`:
```yaml
services:
  homeassistant:
    image: ghcr.io/home-assistant/home-assistant:stable
    container_name: homeassistant
    environment:
      TZ: Europe/Warsaw
    security_opt:
      - no-new-privileges:true
    ports:
      - 8123:8123
    networks:
      - caddy_private
      - smarthome
    volumes:
      - /srv/server/services/smarthome/homeassistant:/config
      - /etc/localtime:/etc/localtime:ro
    restart: unless-stopped

  mosquitto:
    image: eclipse-mosquitto:latest
    container_name: mosquitto
    security_opt:
      - no-new-privileges:true
    networks:
      - smarthome
    volumes:
      - /srv/server/services/smarthome/mosquitto/config:/mosquitto/config
      - /srv/server/services/smarthome/mosquitto/data:/mosquitto/data
      - /srv/server/services/smarthome/mosquitto/log:/mosquitto/log
    restart: unless-stopped

  zigbee2mqtt:
    image: koenkk/zigbee2mqtt:latest
    container_name: zigbee2mqtt
    depends_on:
      - mosquitto
    environment:
      TZ: Europe/Warsaw
    security_opt:
      - no-new-privileges:true
    ports:
      - 8086:8080
    networks:
      - smarthome
    volumes:
      - /srv/server/services/smarthome/zigbee2mqtt/data:/app/data
    restart: unless-stopped

networks:
  caddy_private:
    external: true
  smarthome:
    name: smarthome
```

Notes:
- Home Assistant is reachable through Caddy on `caddy_private`.
- Home Assistant is also exposed on the host at port `8123` for LAN/VPN access.
- Mosquitto is reachable only inside the `smarthome` network.
- Zigbee2MQTT communicates with Mosquitto through the `smarthome` network.
- Zigbee2MQTT frontend is exposed on the host at port `8086` for LAN/VPN access only.
- Do not port forward `8123` or `8086` to the public internet.

---

## Caddy

Example private route:
```caddyfile
@homeassistant host home.lan.{env.BASE_URL}
handle @homeassistant {
    @private client_ip LAN_SUBNET VPN_SUBNET

    handle @private {
        reverse_proxy homeassistant:8123
    }

    respond "Forbidden" 403
}
```

Replace:
- `LAN_SUBNET` with your local network subnet, for example `192.168.20.0/24`
- `VPN_SUBNET` with your VPN subnet, for example `10.8.0.0/24`

Do not expose through Caddy:
- Mosquitto
- Zigbee2MQTT frontend
- SLZB-06MU panel

---

## Home Assistant proxy config

If Home Assistant is behind Caddy, add this to:
```txt
/srv/server/services/smarthome/homeassistant/configuration.yaml
```

Example:
```yaml
http:
  use_x_forwarded_for: true
  trusted_proxies:
    - 172.16.2.0/24
    - 172.16.14.0/24
```

Check Caddy Docker networks:
```bash
sudo docker network inspect caddy | grep -E '"Subnet"|IPv4Address'
sudo docker network inspect caddy_private | grep -E '"Subnet"|IPv4Address'
```

Do not use:
```yaml
trusted_proxies:
  - 0.0.0.0/0
```

---

## Backup

Backup:
```txt
/srv/server/services/smarthome/homeassistant
/srv/server/services/smarthome/mosquitto/config
/srv/server/services/smarthome/mosquitto/data
/srv/server/services/smarthome/zigbee2mqtt/data
```

Important files:
- `homeassistant/configuration.yaml`
- `mosquitto/config/mosquitto.conf`
- `mosquitto/config/passwordfile`
- `zigbee2mqtt/data/configuration.yaml`
- `zigbee2mqtt/data/database.db`

> [!IMPORTANT]
> Do not commit `mosquitto/config/passwordfile` to a public repository.

---

## Notes

- Keep Mosquitto private.
- Keep Zigbee2MQTT private.
- Keep the SLZB-06MU web panel private.
- Use Home Assistant native authentication and 2FA.
- Use LAN/VPN access for administration.
- Keep coordinator firmware updated.
- Keep Zigbee2MQTT and Home Assistant updated.
- Do not expose MQTT or Zigbee administration panels to the public internet.
