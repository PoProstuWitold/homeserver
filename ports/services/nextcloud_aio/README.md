# Nextcloud All-In-One
The official Nextcloud installation method. Provides easy deployment and maintenance with most features included in this one Nextcloud instance.

> **IMPORTANT!** This is rather complicated setup as it can spawn as many as 11 containers. I **really** recommend to check **[offical documentation](https://github.com/nextcloud/all-in-one)** first!

For example if you are planning to use ***Nextcloud Talk*** you must open TCP & UDP port of your choice, but **NOT** any [well known ports](https://en.wikipedia.org/wiki/List_of_TCP_and_UDP_port_numbers), that is those ranging between 0 and 1023.

``docker-compose.yml``
```yaml
services:
  nextcloud-aio-mastercontainer:
    image: nextcloud/all-in-one:latest
    init: true
    restart: unless-stopped
    container_name: nextcloud-aio-mastercontainer # This line is not allowed to be changed as otherwise AIO will not work correctly
    volumes:
      - nextcloud_aio_mastercontainer:/mnt/docker-aio-config # This line is not allowed to be changed as otherwise the built-in backup solution will not work
      - /var/run/docker.sock:/var/run/docker.sock:ro
    ports:
      - 8080:8080
    environment:
      AIO_DISABLE_BACKUP_SECTION: false
      APACHE_PORT: 11000
      APACHE_IP_BINDING: 0.0.0.0
      BORG_RETENTION_POLICY: --keep-within=7d --keep-weekly=4 --keep-monthly=6
      COLLABORA_SECCOMP_DISABLED: false
      NEXTCLOUD_DATADIR: /srv/server/services/nextcloud/data
      NEXTCLOUD_MOUNT: /mnt/
      NEXTCLOUD_UPLOAD_LIMIT: 10G
      NEXTCLOUD_MAX_TIME: 3600
      NEXTCLOUD_MEMORY_LIMIT: 2048M
      NEXTCLOUD_STARTUP_APPS: deck twofactor_totp tasks calendar contacts notes
      NEXTCLOUD_ADDITIONAL_APKS: imagemagick
      NEXTCLOUD_ADDITIONAL_PHP_EXTENSIONS: imagick
      NEXTCLOUD_ENABLE_DRI_DEVICE: true
      NEXTCLOUD_KEEP_DISABLED_APPS: false
      TALK_PORT: 3478
      WATCHTOWER_DOCKER_SOCKET_PATH: /var/run/docker.sock
      SKIP_DOMAIN_VALIDATION: true
    networks:
      - caddy
      - nextcloud-aio
    security_opt: ["label:disable"] # Is needed when using SELinux


volumes:
  nextcloud_aio_mastercontainer:
    name: nextcloud_aio_mastercontainer # This line is not allowed to be changed as otherwise the built-in backup solution will not work

networks:
  caddy:
    name: caddy
    external: true
  nextcloud-aio:
    name: nextcloud-aio # This line is not allowed to be changed as otherwise the created network will not be used by the other containers of AIO
    external: true
```