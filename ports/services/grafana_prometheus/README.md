# Grafana, Prometheus, Loki & Observability

**Grafana** is an observability and visualization platform for metrics, logs and dashboards.

**Prometheus** collects metrics from exporters and services.

**Loki** stores logs.

**Grafana Alloy** reads log files and sends them to Loki.

---

This setup gives you:

- server metrics from **Node Exporter**,
- Docker container metrics from **cAdvisor**,
- disk S.M.A.R.T. metrics from **smartctl_exporter**,
- reverse proxy metrics from **Caddy**,
- security metrics from **CrowdSec**,
- Caddy access logs in Grafana through **Loki + Alloy**.

> [!TIP]
> This setup is intended for a single homeserver. It deliberately avoids the large distributed Loki setup with MinIO, read/write/backend nodes and gateway routing.

---

## Table of contents

<details>
<summary>Click to expand</summary>

- [Directory layout](#directory-layout)
- [Grafana](#grafana)
- [Monitoring stack](#monitoring-stack)
- [Required directories and permissions](#required-directories-and-permissions)
- [Prometheus configuration](#prometheus-configuration)
- [Loki configuration](#loki-configuration)
- [Alloy configuration](#alloy-configuration)
- [Caddy access logs](#caddy-access-logs)
- [CrowdSec log acquisition](#crowdsec-log-acquisition)
- [Starting the stack](#starting-the-stack)
- [Grafana data sources](#grafana-data-sources)
- [Recommended dashboards](#recommended-dashboards)
- [Troubleshooting](#troubleshooting)

</details>

---

## Directory layout

Recommended host-side layout:

```text
/srv/server/services/
├── grafana/
├── prometheus/
│   ├── prometheus.yml
│   └── data/
├── loki/
│   ├── loki-config.yml
│   └── data/
├── alloy/
│   └── config.alloy
├── caddy/
│   ├── Caddyfile
│   └── logs/
│       ├── caddy.log
│       └── caddy-access.log
└── crowdsec/
    └── acquis.yaml
```

> [!IMPORTANT]
> Create configuration files before starting Docker Compose.  
> If a bind-mounted file does not exist, Docker can create a directory with that name, which causes annoying mount errors.

[Back to top](#table-of-contents)

---

## Grafana

Grafana can be kept in a separate stack, especially if it is already attached to the external Caddy network.

`docker-compose.yml`

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

Create the Grafana directory:

```bash
sudo mkdir -p /srv/server/services/grafana
sudo chown -R "$UID:$GID" /srv/server/services/grafana
```

Start Grafana:

```bash
docker compose up -d
```

[Back to top](#table-of-contents)

---

## Monitoring stack

This stack contains:

* Prometheus,
* Node Exporter,
* cAdvisor,
* smartctl_exporter,
* Loki,
* Alloy.

`docker-compose.yml`

```yaml
services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - 9092:9090
    user: "$UID:$GID"
    extra_hosts:
      - "host.docker.internal:host-gateway"
    volumes:
      - /srv/server/services/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - /srv/server/services/prometheus/data:/prometheus
    restart: unless-stopped

  node_exporter:
    image: quay.io/prometheus/node-exporter:latest
    container_name: node_exporter
    command:
      - "--path.rootfs=/host"
    network_mode: host
    pid: host
    volumes:
      - "/:/host:ro,rslave"
    restart: unless-stopped

  cadvisor:
    image: ghcr.io/google/cadvisor:latest
    container_name: cadvisor
    privileged: true
    devices:
      - /dev/kmsg:/dev/kmsg
    ports:
      - 8083:8080
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker:/var/lib/docker:ro
      - /dev/disk:/dev/disk:ro
    restart: unless-stopped

  smartctl_exporter:
    image: prometheuscommunity/smartctl-exporter:latest
    container_name: smartctl_exporter
    privileged: true
    user: root
    ports:
      - 9633:9633
    volumes:
      - /dev:/dev:ro
      - /run/udev:/run/udev:ro
    restart: unless-stopped

  loki:
    image: grafana/loki:latest
    container_name: loki
    ports:
      - 3100:3100
    volumes:
      - /srv/server/services/loki/loki-config.yml:/etc/loki/loki-config.yml:ro
      - /srv/server/services/loki/data:/loki:Z
    command:
      - "-config.file=/etc/loki/loki-config.yml"
    restart: unless-stopped

  alloy:
    image: grafana/alloy:latest
    container_name: alloy
    ports:
      - 12345:12345
    volumes:
      - /srv/server/services/alloy/config.alloy:/etc/alloy/config.alloy:ro
      - /srv/server/services/caddy/logs:/var/log/caddy:ro
    command:
      - "run"
      - "--server.http.listen-addr=0.0.0.0:12345"
      - "--storage.path=/var/lib/alloy/data"
      - "/etc/alloy/config.alloy"
    restart: unless-stopped
```

> [!NOTE]
> `cadvisor` is exposed as `8083:8080` only for optional browser access.  
> Prometheus should scrape it internally as `cadvisor:8080`.

> [!TIP]
> `node_exporter` uses `network_mode: host` because it should monitor the actual host, not just its own container.  
> Because of that, Prometheus reaches it through `host.docker.internal:9100`.

[Back to top](#table-of-contents)

---

## Required directories and permissions

Create directories:

```bash
sudo mkdir -p /srv/server/services/prometheus/data
sudo mkdir -p /srv/server/services/loki/data
sudo mkdir -p /srv/server/services/alloy
```

Create empty config files before starting containers:

```bash
sudo touch /srv/server/services/prometheus/prometheus.yml
sudo touch /srv/server/services/loki/loki-config.yml
sudo touch /srv/server/services/alloy/config.alloy
```

Set permissions:

```bash
sudo chown -R "$UID:$GID" /srv/server/services/prometheus
sudo chown -R 10001:10001 /srv/server/services/loki/data
```

Prepare Loki data directories:

```bash
sudo mkdir -p /srv/server/services/loki/data/chunks
sudo mkdir -p /srv/server/services/loki/data/rules
sudo mkdir -p /srv/server/services/loki/data/compactor

sudo chown -R 10001:10001 /srv/server/services/loki/data
sudo chmod -R u+rwX /srv/server/services/loki/data
```

> [!IMPORTANT]
> Loki commonly runs as UID `10001`.  
> If `/srv/server/services/loki/data` is owned by `root`, Loki may fail with `permission denied`.

> [!NOTE]
> The `:Z` suffix on the Loki data volume is useful on SELinux systems.  
>
> If your system does not use SELinux, it is usually harmless or unnecessary.

[Back to top](#table-of-contents)

---

## Prometheus configuration

File:

```text
/srv/server/services/prometheus/prometheus.yml
```

`prometheus.yml`

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: "CrowdSec"
    static_configs:
      - targets:
          - "crowdsec-or-hostname:6060"
        labels:
          machine: "SERVER_NAME"

  - job_name: "Caddy"
    static_configs:
      - targets:
          - "metrics.your-domain.tld"
        labels:
          machine: "SERVER_NAME"

  - job_name: "NodeExporter"
    static_configs:
      - targets:
          - "host.docker.internal:9100"
        labels:
          machine: "SERVER_NAME"

  - job_name: "cAdvisor"
    static_configs:
      - targets:
          - "cadvisor:8080"
        labels:
          machine: "SERVER_NAME"

  - job_name: "smartctl"
    static_configs:
      - targets:
          - "smartctl_exporter:9633"
        labels:
          machine: "SERVER_NAME"
```

> [!WARNING]
> Job names are case-sensitive.  
> If a dashboard expects `cAdvisor`, then `cadvisor` is not the same thing.

[Back to top](#table-of-contents)

---

## Loki configuration

File:

```text
/srv/server/services/loki/loki-config.yml
```

`loki-config.yml`

```yaml
auth_enabled: false

server:
  http_listen_port: 3100

common:
  path_prefix: /loki
  storage:
    filesystem:
      chunks_directory: /loki/chunks
      rules_directory: /loki/rules
  replication_factor: 1
  ring:
    kvstore:
      store: inmemory

schema_config:
  configs:
    - from: 2024-01-01
      store: tsdb
      object_store: filesystem
      schema: v13
      index:
        prefix: index_
        period: 24h

limits_config:
  retention_period: 168h
  allow_structured_metadata: false

compactor:
  working_directory: /loki/compactor
  retention_enabled: true
  delete_request_store: filesystem
```

> [!TIP]
> `retention_period: 168h` means Loki keeps logs for 7 days.

[Back to top](#table-of-contents)

---

## Alloy configuration

Alloy reads Caddy access logs and sends them to Loki.

File:

```text
/srv/server/services/alloy/config.alloy
```

`config.alloy`

```hcl
loki.write "default" {
    endpoint {
        url = "http://loki:3100/loki/api/v1/push"
    }
}

local.file_match "caddy_access_logs" {
    path_targets = [
        {
            __path__ = "/var/log/caddy/caddy-access.log",
            job = "caddy-access",
            machine = "SERVER_NAME",
        },
    ]
}

loki.source.file "caddy_access_logs" {
    targets = local.file_match.caddy_access_logs.targets
    forward_to = [loki.process.caddy_access.receiver]
}

loki.process "caddy_access" {
    stage.json {
        expressions = {
            ts = "ts",
            level = "level",
            logger = "logger",
            host = "request.host",
            method = "request.method",
            uri = "request.uri",
            status = "status",
            duration = "duration",
            client_ip = "request.remote_ip",
        }
    }

    stage.labels {
        values = {
            host = "",
            method = "",
            status = "",
        }
    }

    forward_to = [loki.write.default.receiver]
}
```

> [!IMPORTANT]
> Do not use `uri` or `client_ip` as Loki labels.
>  
> These values can create very high cardinality. Keep them as parsed fields, not labels.

[Back to top](#table-of-contents)

---

## Caddy access logs

The target log files are:

```text
/var/log/caddy/caddy.log
/var/log/caddy/caddy-access.log
```

Recommended meaning:

|File|Purpose|
|-|-|
|`caddy.log`|Caddy runtime logs, TLS logs, startup logs, config errors|
|`caddy-access.log`|HTTP access logs for Loki, Grafana and CrowdSec|

### Global logging block

Inside the global Caddy block:

```caddyfile
{
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

        crowdsec {
                api_url http://crowdsec:8080
                api_key {env.CROWDSEC_API_KEY}
        }

        order crowdsec first

        servers {
                trusted_proxies cloudflare
                client_ip_headers CF-Connecting-IP
        }
}
```

### Enable access logging for sites

Access logging must be enabled in a site block or in a snippet imported by a site block.

Example snippet:

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
```

> [!IMPORTANT]
> The line `log` is required.
> 
> The global logger with `include http.log.access` only redirects access logs. It does not enable access logging by itself.

### Caddy Docker volume

The Caddy container needs the logs directory mounted:

```yaml
volumes:
  - /srv/server/services/caddy/logs:/var/log/caddy
```

> [!WARNING]
> Do not keep Caddy `debug` enabled permanently.  
> Debug logs can include very verbose request data and potentially sensitive headers.

[Back to top](#table-of-contents)

---

## CrowdSec log acquisition

File:

```text
/srv/server/services/crowdsec/acquis.yaml
```

`acquis.yaml`

```yaml
filenames:
  - /var/log/caddy/*.log
labels:
  type: caddy
```

CrowdSec should have Caddy logs mounted read-only:

```yaml
volumes:
  - /srv/server/services/caddy/logs:/var/log/caddy:ro
```

[Back to top](#table-of-contents)

---

## Starting the stack

Start or redeploy:

```bash
docker compose up -d
```

Check containers:

```bash
docker ps
```

Check Prometheus:

```bash
curl http://YOUR_SERVER_IP:9092/-/ready
```

Check Loki:

```bash
curl http://YOUR_SERVER_IP:3100/ready
```

Check Alloy logs:

```bash
docker logs alloy --tail=100
```

Check Loki logs:

```bash
docker logs loki --tail=100
```

Check if Alloy sees Caddy logs:

```bash
docker exec -it alloy ls -lah /var/log/caddy
```

[Back to top](#table-of-contents)

---

## Grafana data sources

### Prometheus

Add Prometheus:

```text
Connections -> Data sources -> Add data source -> Prometheus
```

URL if Grafana can resolve the Prometheus container:

```text
http://prometheus:9090
```

URL if Grafana is in another Docker network:

```text
http://YOUR_SERVER_IP:9092
```

### Loki

Add Loki:

```text
Connections -> Data sources -> Add data source -> Loki
```

URL if Grafana can resolve the Loki container:

```text
http://loki:3100
```

URL if Grafana is in another Docker network:

```text
http://YOUR_SERVER_IP:3100
```

> [!NOTE]
> If Grafana is in a different Docker Compose stack, either use the host-exposed ports or connect Grafana to the same Docker network as Prometheus/Loki.

[Back to top](#table-of-contents)

---

## Recommended dashboards

Import dashboards by ID:

|Dashboard|ID|
|-|-:|
|Node Exporter Full|`1860`|
|cAdvisor / Docker containers|`19908`|
|smartctl_exporter|`22381`|
|Caddy Monitoring with Prometheus + Loki|`22806`|

Import path:

```text
Dashboards -> New -> Import -> paste dashboard ID -> Load
```

When importing, choose the right data sources:

```text
Prometheus -> your Prometheus data source
Loki -> your Loki data source
```

For this setup, common values are:

```text
Caddy metrics job: Caddy
cAdvisor job: cAdvisor
Node Exporter job: NodeExporter
smartctl job: smartctl
Loki access log job: caddy-access
```

> [!WARNING]
> Some ready-made Caddy dashboards expect Prometheus metrics that your Caddy build may not expose.
>
> If top panels show `No data`, but Loki log panels work, replace those panels with LogQL queries based on `{job="caddy-access"}`.

[Back to top](#table-of-contents)

---

## Troubleshooting

### Loki: permission denied

Example error:

```text
mkdir /loki/rules: permission denied
```

Fix:

```bash
docker stop loki

sudo mkdir -p /srv/server/services/loki/data/chunks
sudo mkdir -p /srv/server/services/loki/data/rules
sudo mkdir -p /srv/server/services/loki/data/compactor

sudo chown -R 10001:10001 /srv/server/services/loki/data
sudo chmod -R u+rwX /srv/server/services/loki/data

docker compose up -d
```

If SELinux is enabled, use:

```yaml
- /srv/server/services/loki/data:/loki:Z
```

---

### Prometheus cannot resolve cAdvisor

Example error:

```text
lookup cadvisor on 127.0.0.11:53: no such host
```

Make sure Prometheus and cAdvisor are in the same Compose stack or Docker network.

Prometheus should scrape cAdvisor using the internal container port:

```yaml
- "cadvisor:8080"
```

Even if cAdvisor is exposed on the host as:

```yaml
ports:
  - "8083:8080"
```

---

### Node Exporter is down

If Node Exporter runs with:

```yaml
network_mode: host
```

Prometheus should scrape it using:

```yaml
- "host.docker.internal:9100"
```

Prometheus needs:

```yaml
extra_hosts:
  - "host.docker.internal:host-gateway"
```

---

### Caddy access log is empty

Make sure access logging is enabled in the site block or imported snippet:

```caddyfile
log
```

Make sure the global access logger exists:

```caddyfile
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
```

Restart Caddy:

```bash
docker restart caddy
```

Generate traffic:

```bash
curl https://dashdot.your-domain.tld >/dev/null
```

Check the access log:

```bash
sudo tail -n 5 /srv/server/services/caddy/logs/caddy-access.log
```

Check active Caddy config:

```bash
docker exec caddy caddy validate --config /etc/caddy/Caddyfile
docker exec caddy caddy adapt --config /etc/caddy/Caddyfile | grep -n "http.log.access" -C 5
docker logs caddy --tail=100
```

Format Caddyfile:

```bash
docker exec caddy caddy fmt --overwrite /etc/caddy/Caddyfile
docker exec caddy caddy validate --config /etc/caddy/Caddyfile
docker restart caddy
```

---

### Alloy does not send logs to Loki

Check if Alloy can see Caddy logs:

```bash
docker exec -it alloy ls -lah /var/log/caddy
```

Check Alloy logs:

```bash
docker logs alloy --tail=100
```

Check Loki readiness:

```bash
curl http://localhost:3100/ready
```

Then test in Grafana Explore:

```logql
{job="caddy-access"}
```

---

### Ready-made dashboard shows partial data

If Loki log panels work but some top panels show `No data`, the dashboard probably expects different Prometheus Caddy metric names.

Use Loki-based panels instead:

```logql
sum(count_over_time({job="caddy-access"}[$__range]))
```

```logql
sum by (status) (count_over_time({job="caddy-access"}[$__range]))
```

```logql
sum by (host) (count_over_time({job="caddy-access"}[$__range]))
```

```logql
quantile_over_time(0.95, {job="caddy-access"} | unwrap duration [$__range])
```

[Back to top](#table-of-contents)

---

## Notes

> [!TIP]
> Keep runtime logs and access logs separate. It makes Grafana, Loki and CrowdSec much cleaner.

> [!WARNING]
> Do not keep Caddy `debug` enabled permanently. Debug logs can include sensitive headers or tokens.

> [!NOTE]
> Ready-made Grafana dashboards often need small query and label adjustments. Your real labels are the source of truth.

