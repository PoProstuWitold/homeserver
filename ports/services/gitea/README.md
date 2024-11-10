# Gitea
Git with a cup of tea! Painless self-hosted all-in-one software development service, including Git hosting, code review, team collaboration, package registry and CI/CD.

After you launch your ``docker-compose.yml`` run these commands in main ``gitea`` catalog to fix issues with permissions:
``sudo chown 1000:1000 ./config/``
``sudo chown 1000:1000 ./data/``
``sudo chown 1000:1000 ./db/``
``sudo chmod -R 755 ./db ``

You also have to modify ``config/app.ini`` file to looks similar to this:
```ini
[server]
APP_DATA_PATH = /var/lib/gitea                                                                                          PROTOCOL PROTOCOL = http
SSH_DOMAIN = gitea.yourdomain.tld
ROOT_URL = https://gitea.yourdomain.tld
DISABLE_SSH = false
; In rootless gitea container only internal ssh server is supported
START_SSH_SERVER = true
SSH_PORT = 2222
SSH_LISTEN_PORT = 2222
BUILTIN_SSH_SERVER_USER = YOUR_USERNAME ; of UID:GID (1000:1000)
LFS_START_SERVER = true
DOMAIN = localhost
LFS_JWT_SECRET = GENERATED_SECRET
OFFLINE_MODE = true 
```

``docker-compose.yml``
```yaml
services:
  gitea:
    container_name: gitea
    image: gitea/gitea:1.22.3-rootless
    environment:
    # DB
      - GITEA__database__DB_TYPE=postgres
      - GITEA__database__HOST=gitea_db:5432
      - GITEA__database__NAME=gitea
      - GITEA__database__USER=gitea
      - GITEA__database__PASSWD=CHANGE_ME
    # OTHER
      - APP_NAME="Gitea - Git with a cup of tea" 
      - USER=YOUR_USERNAME # of UID:GID (1000:1000)
    restart: always
    volumes:
      - /srv/server/services/gitea/data:/var/lib/gitea
      - /srv/server/services/gitea/config:/etc/gitea
      - /etc/timezone:/etc/timezone:ro
      - /etc/localtime:/etc/localtime:ro
    ports:
      - "3004:3000"
      - "2222:2222"
    depends_on:
      - gitea_db
    networks:
      - gitea
      - caddy

  gitea_db:
    container_name: gitea_db
    image: postgres:14
    restart: always
    environment:
      - POSTGRES_USER=gitea
      - POSTGRES_PASSWORD=CHANGE_ME
      - POSTGRES_DB=gitea
    volumes:
      - /srv/server/services/gitea/db:/var/lib/postgresql/data
    networks:
      - gitea

networks:
  caddy:
    name: caddy
    external: true
  gitea:
    name: gitea
```