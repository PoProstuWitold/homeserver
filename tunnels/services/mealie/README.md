# Mealie
[Mealie](https://github.com/mealie-recipes/mealie) is a self hosted recipe manager and meal planner for a pleasant user experience for the whole family. It's basically your recipes library where you can add, remove, import or export recipes with ease

``docker-compose.yml``
```yaml
services:
  mealie:
    image: hkotel/mealie:latest
    container_name: mealie
    ports:
      - 9091:9000
    deploy:
      resources:
        limits:
          memory: 1000M 
    environment:
      PUID: 1000
      PGID: 1000
      TZ: Europe/Warsaw
      BASE_URL: https://mealie.your-domain.tld
    volumes:
      - /srv/server/services/mealie:/app/data
    restart: unless-stopped
```


