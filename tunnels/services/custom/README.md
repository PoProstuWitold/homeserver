# Custom service
You can also run your own apps using something like **[pm2](https://pm2.keymetrics.io/)** or **[docker compose](https://docs.docker.com/compose/)**. You can even combine both as I'm about to show you.

> You can also just use ``pm2`` without docker

First you need app that you want to selfhosted. It can be really anything but probably it will be some web server, fullstack app or bot.

For this example I will use my discord bot ***[Sayuna](https://github.com/PoProstuWitold/Sayuna)***. It already have docker image and docker compose file which look like this:


``Dockerfile``
```dockerfile
### build
FROM node:lts AS setup

WORKDIR /app

COPY package.json pnpm-lock.yaml ./
COPY . .

RUN apt-get update && apt-get install -y ffmpeg
RUN npm install -g pnpm
RUN pnpm install --frozen-lockfile


### development
FROM setup AS development

WORKDIR /app

CMD [ "pnpm", "run", "dev" ]

### production
FROM setup AS production

WORKDIR /app

COPY --from=setup /app .

RUN npm install -g pm2

ENV BOT_TOKEN=$BOT_TOKEN \
    DEV_GUILD_ID=$DEV_GUILD_ID \
    OWNER_ID=$OWNER_ID \
    BOT_ID=$BOT_ID \
    BOT_PREFIX=$BOT_PREFIX \
    AI_ENABLED=$AI_ENABLED \
    CHAT_GPT_API_KEY=$CHAT_GPT_API_KEY \

RUN pnpm run build
CMD ["pm2-runtime", "build/main.js"]
```

``docker-compose.yml``
```yaml
services:
  sayuna:
    container_name: Sayuna
    build:
      context: .
      target: production
    stdin_open: true
    tty: true
    env_file:
      .env
    restart: unless-stopped
```

As you can see I use ``pm2`` inside my docker container. You can also expose ports if your app needs it. 

Now in order to run this, place ``.env`` file and all ``Sayuna`` files in same directory and just run:

```bash
docker compose up -d
```

This will run your app in the background. You will be able to control it through ``Portainer``.