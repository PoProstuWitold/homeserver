# SFTPGo
Fully featured and highly configurable SFTP server with HTTP/S Web UI and optional FTP/S and WebDAV.

## Create required directories
```bash
mkdir -p /srv/server/services/sftpgo/data
mkdir -p /srv/server/services/sftpgo/config
```

## Fix permissions
```bash
chown -R 1000:1000 /srv/server/services/sftpgo/data
chown -R 1000:1000 /srv/server/services/sftpgo/config
```

## Setting a Root Directory for Files (in the Web UI)
When configuring a virtual folder or user's root directory in the SFTPGo admin panel, use the path inside the container, not the host path.

For example:
```bash
/var/lib/sftpgo/files
```

Maps to:
```bash
/srv/server/services/sftpgo/data/files
```

So create the folder on your host and fix its permissions:
```bash
mkdir -p /srv/server/services/sftpgo/data/files
chown -R 1000:1000 /srv/server/services/sftpgo/data/files
```

Then enter ``/var/lib/sftpgo/files`` in the “Root directory” field in the web UI.

> When creating a user, set their home directory to the virtual folder path followed by their ``username``, for example: 
> ``/var/lib/sftpgo/files/username``

``docker-compose.yml``
```yaml
services:
  sftpgo:
    image: drakkan/sftpgo:latest
    container_name: sftpgo
    ports:
      - 2022:2022
      - 8080:8080
    networks:
      - caddy
    volumes:
      - /srv/server/services/sftpgo/data:/var/lib/sftpgo
      - /srv/server/services/sftpgo/config:/etc/sftpgo
    restart: unless-stopped

networks:
  caddy:
    name: caddy
    external: true
```