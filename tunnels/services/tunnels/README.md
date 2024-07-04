# Cloudflare Tunnels
Launch your tunneling as docker container.

Remember to update ``random-token`` to your Cloudflare Tunnel token.

``docker-compose.yml``
```yaml
version: "3.8"

services:
  cloudflared:
    image: cloudflare/cloudflared:latest
    command: tunnel --no-autoupdate run --token random-token
    restart: unless-stopped
```