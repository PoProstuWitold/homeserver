# Grafana & Prometheus

**[Grafana](https://github.com/grafana/grafana)** is an open and composable observability and data visualization platform. Visualize metrics, logs, and traces from multiple sources like Prometheus, Loki, Elasticsearch, InfluxDB, Postgres and many more.

**[Prometheus]()** is monitoring system and time series database. One of possible data source for [Grafana dashboards](https://grafana.com/grafana/dashboards/).


## Grafana

All you need is ``docker-compose.yml`` file below:

``docker-compose.yml``
```yaml
services:
  grafana:
    image: grafana/grafana-oss:latest
    container_name: grafana
    environment:
     - GF_SERVER_ROOT_URL=https://grafana.your-domain.tld/
     - GF_INSTALL_PLUGINS=grafana-clock-panel
    ports:
     - 3003:3000
    user: "$UID:$GID"
    networks:
     - caddy
    volumes:
     - /srv/server/services/grafana:/var/lib/grafana
    restart: unless-stopped

networks:
  caddy:
    name: caddy
    external: true
```

## Prometheus

For this service you need to also edit ``prometheus.yml`` file:

``prometheus.yml``
```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: "CrowdSec"
    static_configs:
    - targets: ["IPv4_FQDM_or_CONTAINER_NAME_IF_IN_SAME_NETWORK:6060"]
      labels:
        machine: "machine name"

  - job_name: "Caddy"
    static_configs:
    - targets: ["metrics.your-domain.tld"]
      labels:
        machine: "machine name"
```

``docker-compose.yml``
```yaml
services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - 9092:9090
    user: "$UID:$GID"
    volumes:
      - /srv/server/services/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - /srv/server/services/prometheus/data:/prometheus
```