# Home Assistant + Zigbee2MQTT + Mosquitto + SLZB-06MU

> [!WARNING]
> These instructions are currently a work in progress.
>
> The setup described here has not been fully tested in my own environment yet.
>
> Some configuration values, paths, ports, or device-specific settings may still require adjustments after real hardware testing.
>
> **Use this as a draft reference, not as a final production-ready guide.**

Local smart home stack based on:
- Docker
- Home Assistant
- Zigbee2MQTT
- Mosquitto MQTT
- SMLIGHT SLZB-06MU

Everything runs locally without depending on vendor cloud services.

---

## Table of Contents

- [Architecture](#architecture)
- [Requirements](#requirements)
- [Directory Structure](#directory-structure)
- [Mosquitto MQTT Configuration](#mosquitto-mqtt-configuration)
- [Zigbee2MQTT Configuration](#zigbee2mqtt-configuration)
- [Docker Compose](#docker-compose)
- [Home Assistant](#home-assistant)
- [Pairing Zigbee Devices](#pairing-zigbee-devices)

---

## Architecture

Zigbee devices connect wirelessly to the SMLIGHT SLZB-06MU coordinator.

The coordinator is connected to the local network via Ethernet and is used by Zigbee2MQTT to communicate with Zigbee devices.

Zigbee2MQTT sends device data to Mosquitto MQTT, and Home Assistant reads that data from MQTT.

This keeps the whole smart home setup local, without relying on vendor cloud services.

---

## Requirements

- Linux server / homelab
- Docker
- Docker Compose
- Router
- Ethernet
- SMLIGHT SLZB-06MU
- USB-C power adapter for SMLIGHT SLZB-06MU

---

## Directory Structure

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

[Back to top](#table-of-contents)

---

## Mosquitto MQTT Configuration

Create the config file:

```bash
sudo nano /srv/server/services/smarthome/mosquitto/config/mosquitto.conf
```

Configuration:

```txt
persistence true
persistence_location /mosquitto/data/

listener 1883
allow_anonymous true
```

[Back to top](#table-of-contents)

---

## Zigbee2MQTT Configuration

Create the config file:

```bash
sudo nano /srv/server/services/smarthome/zigbee2mqtt/data/configuration.yaml
```

Configuration:

```yaml
homeassistant:
  enabled: true

frontend:
  enabled: true
  port: 8080

mqtt:
  server: mqtt://mosquitto:1883

serial:
  port: tcp://SLZB_06MU_IP:6638

advanced:
  channel: 15

permit_join: false
```

> [!IMPORTANT]
> Replace `SLZB_06MU_IP` with the IP address of your SMLIGHT SLZB-06MU coordinator. **This is not the server IP address.**

SLZB-06MU connections:
- Ethernet → router
- USB-C → power adapter

[Back to top](#table-of-contents)

---

## Docker Compose

`docker-compose.yml`

```yaml
services:
  homeassistant:
    image: ghcr.io/home-assistant/home-assistant:stable
    container_name: homeassistant
    privileged: true
    network_mode: host
    environment:
      TZ: Europe/Warsaw
    volumes:
      - /srv/server/services/smarthome/homeassistant:/config
      - /etc/localtime:/etc/localtime:ro
    restart: unless-stopped

  mosquitto:
    image: eclipse-mosquitto:latest
    container_name: mosquitto
    ports:
      - 1883:1883
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
    ports:
      - 8080:8080
    environment:
      TZ: Europe/Warsaw
    networks:
      - smarthome
    volumes:
      - /srv/server/services/smarthome/zigbee2mqtt/data:/app/data
    restart: unless-stopped

networks:
  smarthome:
    name: smarthome
```

[Back to top](#table-of-contents)

---

## Home Assistant

Home Assistant:

```txt
http://SERVER_IP:8123
```

Zigbee2MQTT:

```txt
http://SERVER_IP:8080
```

SLZB-06MU:

```txt
http://SLZB_06MU_IP
```

Add MQTT integration:

```txt
Settings
→ Devices & services
→ Add integration
→ MQTT
```

Configuration:

```txt
Broker:
127.0.0.1

Port:
1883
```

If it does not work:

```txt
Broker:
SERVER_IP
```

[Back to top](#table-of-contents)

---

## Pairing Zigbee Devices

Edit:

```bash
sudo nano /srv/server/services/smarthome/zigbee2mqtt/data/configuration.yaml
```

Change:

```yaml
permit_join: true
```

Restart:

```bash
sudo docker restart zigbee2mqtt
```

Open:

```txt
http://SERVER_IP:8080
```

Pair your Zigbee device.

After pairing:

```yaml
permit_join: false
```

Restart again:

```bash
sudo docker restart zigbee2mqtt
```

[Back to top](#table-of-contents)
