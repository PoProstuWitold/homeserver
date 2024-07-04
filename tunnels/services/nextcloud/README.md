# NextCloud
Regain control over your data

Remember to change: ``MYSQL_ROOT_PASSWORD``, ``MYSQL_PASSWORD`` in ``db`` container and ``MYSQL_PASSWORD`` in ``app`` container.

``docker-compose.yml``
```yaml
services:
  db:
    container_name: nextcloud_db
    image: mariadb:latest
    command: --transaction-isolation=READ-COMMITTED --log-bin=binlog --binlog-format=ROW --character-set-server=utf8
    volumes:
      - /srv/server/services/nextcloud/db:/var/lib/mysql
    environment:
      - MYSQL_ROOT_PASSWORD=changeme
      - MYSQL_PASSWORD=changeme
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud
    restart: unless-stopped
  app:
    container_name: nextcloud
    image: nextcloud:latest
    ports:
      - 8080:80
    links:
      - db
    depends_on:
      - db
    volumes:
      - /srv/server/services/nextcloud/nc:/var/www/html
      - /etc/localtime:/etc/localtime:ro
    environment:
      - MYSQL_PASSWORD=changeme
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud
      - MYSQL_HOST=db
      - REDIS_HOST=nextcloud_cache
      - REDIS_PORT=6380
      - REDIS_HOST_PASSWORD=nextcloud_cache_password
    restart: unless-stopped
  redis:
    container_name: nextcloud_cache
    image: redis
    ports:
      - 6380:6379
    command: redis-server --requirepass nextcloud_cache_password
    restart: unless-stopped
```

### How to set up `Cron` as background jobs?
By default **NextCloud** is using `AJAX` requests to run background jobs. However recommended way of doing it is to use `Cron`. It may be tricky to setup but it's trivial when you see it once.

1. Install `cronie` on distro of your choice
2. Run following command:
```bash
sudo crontab -u <YOUR_USERNAME> -e
```
3. Paste there following line and save:
```bash
*/5 * * * * docker exec -u www-data nextcloud php cron.php
```
where `nextcloud` is your Docker container name.

After you did all of above you can change this setting in NextCloud Web UI.