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
      - /home/docker/nextcloud/etc/localtime:/etc/localtime:ro
    environment:
      - MYSQL_PASSWORD=changeme
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud
      - MYSQL_HOST=db
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
*/5 * * * * docker exec -u www-data nextcloud-app-1 php cron.php
```
where `nextcloud-app-1` is your Docker container name.

After you did all of above you can change this setting in NextCloud web UI.