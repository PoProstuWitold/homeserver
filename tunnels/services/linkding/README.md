# Linkding
Self-hosted bookmark manager that is designed be to be minimal, fast, and easy to set up using Docker.

You can get all `env` options [here](https://github.com/sissbruecker/linkding/blob/master/.env.sample)

``docker-compose.yml``
```yaml
version: "3.8"

services:
  linkding:
    container_name: "${LD_CONTAINER_NAME:-linkding}"
    image: sissbruecker/linkding:latest
    ports:
      - "${LD_HOST_PORT:-9090}:9090"
    volumes:
      - "${LD_HOST_DATA_DIR:-/home/docker/linkding}:/etc/linkding/data"
    env_file:
      - stack.env
    restart: unless-stopped
```

