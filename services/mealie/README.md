# Mealie
[Mealie](https://github.com/mealie-recipes/mealie) is a self hosted recipe manager and meal planner for a pleasant user experience for the whole family. It's basically your recipes library where you can add, remove, import or export recipes with ease

``docker-compose.yml``
```yaml
vversion: "3.8"

services:
  mealie:
    container_name: mealie
    image: hkotel/mealie:latest
    restart: always
    ports:
      - 9925:80
    environment:
      PUID: 1000
      PGID: 1000
      TZ: Europe/Warsaw

      # Default Recipe Settings
      RECIPE_PUBLIC: 'true'
      RECIPE_SHOW_NUTRITION: 'true'
      RECIPE_SHOW_ASSETS: 'true'
      RECIPE_LANDSCAPE_VIEW: 'true'
      RECIPE_DISABLE_COMMENTS: 'false'
      RECIPE_DISABLE_AMOUNT: 'false'
    volumes:
      - /home/docker/mealie:/app/data

```


