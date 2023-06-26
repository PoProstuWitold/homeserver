# Dashy
The Ultimate Homepage for your Homelab

``docker-compose.yml``
```yaml
version: "3.8"

services:
  dashy:
    image: lissy93/dashy
    container_name: dashy
    ports:
      - 4000:80
    environment:
      - NODE_ENV=production
    restart: always
    healthcheck:
      test: ['CMD', 'node', '/app/services/healthcheck']
      interval: 1m30s
      timeout: 10s
      retries: 3
      start_period: 40s
```