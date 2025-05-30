# Diun
CLI app to receive notifications when an image is updated on a Docker registry.

``diun.yml``
```yaml
watch:
  workers: 20
  schedule: "0 */6 * * *"
  firstCheckNotif: false

providers:
  docker:
    watchByDefault: true

notif:
  discord:
    webhookURL: YOUR_DISCORD_WEBHOOK_URL
    mentions:
      - "@here"
      - "@everyone"
      - "<@124>"
      - "<@125>"
      - "<@&200>"
    renderFields: true
    timeout: 10s
    templateBody: |
      Docker tag {{ if .Entry.Image.HubLink }}[**{{ .Entry.Image }}**]({{ .Entry.Image.HubLink }}){{ else }}**{{ .Entry.Image }}**{{ end }} which you subscribed to through {{ .Entry.Provider }} provider {{ if (eq .Entry.Status "new") }}is available{{ else }}has been updated{{ end }} on {{ .Entry.Image.Domain }} registry (triggered by {{ .Meta.Hostname }} host).

```

``docker-compose.yml``
```yaml
services:
  diun:
    image: crazymax/diun:latest
    container_name: diun
    command: serve
    volumes:
      - /srv/server/services/diun/data:/data
      - /srv/server/services/diun/diun.yml:/diun.yml:ro
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      - TZ=Europe/Warsaw
      - LOG_LEVEL=info
      - LOG_JSON=false
    restart: unless-stopped
```