# Mealie
[Mealie](https://github.com/mealie-recipes/mealie) is a self hosted recipe manager and meal planner for a pleasant user experience for the whole family. It's basically your recipes library where you can add, remove, import or export recipes with ease

The official documentation recommends setting an explicit memory limit, because Python can pre-allocate larger amounts of memory than is necessary if you have a machine with a lot of RAM. This can cause the container to idle at a high memory usage. 

Setting a memory limit will improve idle performance.

``docker-compose.yml``
```yaml
services:
  mealie:
    container_name: mealie
    image: hkotel/mealie:latest
    ports:
      - 9091:9000
    deploy:
      resources:
        limits:
          memory: 1000M 
    networks:
      - caddy
    environment:
      ALLOW_SIGNUP: true
      PUID: 1000
      PGID: 1000
      TZ: Europe/Warsaw
      BASE_URL: https://mealie.your-domain.tld
    volumes:
      - /srv/server/services/mealie:/app/data
    restart: unless-stopped


networks:
  caddy:
    name: caddy
    external: true
```


