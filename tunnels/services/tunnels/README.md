# Cloudflare Tunnels
Launch your tunneling as docker container.

Remember to update ``YOUR_TOKEN`` to your Cloudflare Tunnel token.

``docker-compose.yml``
```yaml
services:
  cloudflared:
    image: cloudflare/cloudflared:latest
    container_name: cloudflared
    command: tunnel --no-autoupdate run --token YOUR_TOKEN
    restart: unless-stopped
```