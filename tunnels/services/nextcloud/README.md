# NextCloud
Regain control over your data

Remember to change: ``MYSQL_ROOT_PASSWORD``, ``MYSQL_PASSWORD`` in ``db`` container and ``MYSQL_PASSWORD`` in ``app`` container.

``docker-compose.yml``
```yaml
version: "3.8"

volumes:
  nextcloud:
  db:

services:
  db:
    container_name: nextcloud_db
    image: mariadb:latest
    restart: always
    command: --transaction-isolation=READ-COMMITTED --log-bin=binlog --binlog-format=ROW --character-set-server=utf8
    volumes:
      - /home/docker/nextcloud/db:/var/lib/mysql
    environment:
      - MYSQL_ROOT_PASSWORD=changeme
      - MYSQL_PASSWORD=changeme
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud

  app:
    container_name: nextcloud
    image: nextcloud:latest
    restart: always
    ports:
      - 8080:80
    links:
      - db
    depends_on:
      - db
    volumes:
      - /home/docker/nextcloud/nextcloud:/var/www/html
      - /etc/localtime:/etc/localtime:ro
    environment:
      - MYSQL_PASSWORD=changeme
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud
      - MYSQL_HOST=db
      - REDIS_HOST=nextcloud_cache
      - REDIS_PORT=6380
      - REDIS_HOST_PASSWORD=nextcloud_cache_password
    redis:
      container_name: nextcloud_cache
      image: redis
      restart: unless-stopped
      ports:
        - 6380:6379
      command: redis-server --requirepass nextcloud_cache_password
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