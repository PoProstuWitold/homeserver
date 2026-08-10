# Nextcloud All-In-One

[Nextcloud All-In-One](https://github.com/nextcloud/all-in-one) is the official Nextcloud deployment method. The AIO mastercontainer manages the remaining Nextcloud containers, their lifecycle, configuration, updates, and the built-in Borg backup system.

> [!IMPORTANT]
> AIO creates and manages multiple containers automatically.
>
> Do not rename the `nextcloud-aio-mastercontainer` or its configuration volume.

## Docker Compose

`docker-compose.yml`

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
    devices:
    - /dev/dri
    ports:
      - 8081:8080
    environment:
      AIO_DISABLE_BACKUP_SECTION: false
      APACHE_PORT: 11000
      APACHE_IP_BINDING: 0.0.0.0
      BORG_RETENTION_POLICY: --keep-within=7d --keep-weekly=4 --keep-monthly=6
      COLLABORA_SECCOMP_DISABLED: false
      NEXTCLOUD_DATADIR: /srv/server/services/nextcloud/data
      NEXTCLOUD_MOUNT: /srv/server/media
      NEXTCLOUD_UPLOAD_LIMIT: 10G
      NEXTCLOUD_MAX_TIME: 3600
      NEXTCLOUD_MEMORY_LIMIT: 2048M
      NEXTCLOUD_STARTUP_APPS: deck twofactor_totp tasks calendar contacts notes
      NEXTCLOUD_ADDITIONAL_APKS: imagemagick
      NEXTCLOUD_ADDITIONAL_PHP_EXTENSIONS: imagick
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

> [!NOTE]
> Adjust resource limits, startup applications, ports, networks, and host paths to match your deployment.

---

## Data Directory

This setup stores the Nextcloud user data directory directly on the host:

```text
/srv/server/services/nextcloud/data
```

AIO exposes it inside the Nextcloud container as:

```text
/srv/server/services/nextcloud/data -> /mnt/ncdata
```

The directory is managed by Nextcloud and contains the `.ncdata` marker used to identify a valid Nextcloud data directory.

A typical directory may contain:

```text
.ncdata
appdata_*/
files_external/
<user directories>/
```

> [!CAUTION]
> Do not manually recreate `.ncdata` as a workaround for a missing or incorrectly mounted data directory.
>
> First verify that the correct host directory is mounted to `/mnt/ncdata`.

The mount can be inspected with:

```bash
docker inspect nextcloud-aio-nextcloud \
  --format '{{range .Mounts}}{{println .Source "->" .Destination}}{{end}}'
```

---

## Local External Storage

`NEXTCLOUD_MOUNT` exposes an additional host directory to the Nextcloud container:

```yaml
NEXTCLOUD_MOUNT: /srv/server/media
```

With this configuration, the directory is available inside the Nextcloud container under the same path:

```text
/srv/server/media -> /srv/server/media
```

This does **not** make it part of the main Nextcloud data directory. It only makes the host path available for use with Nextcloud's **External storage support** application.

For example, individual directories can be configured as separate local storage mounts:

```text
Anime     -> /srv/server/media/anime
Books     -> /srv/server/media/books
Movies    -> /srv/server/media/movies
Music     -> /srv/server/media/music
TV Shows  -> /srv/server/media/tvshows
```

### Configure External Storage

1. Enable the **External storage support** application in Nextcloud.
2. Open **Administration settings**.
3. Go to **External storage**.
4. Click **Add external storage**.
5. Select **Local**.
6. Enter the absolute path exposed through `NEXTCLOUD_MOUNT`.
7. Restrict access to the appropriate users or groups if required.

Example:

```text
Folder name:       Movies
External storage:  Local
Authentication:    None
Configuration:     /srv/server/media/movies
```

> [!TIP]
> If the files are primarily managed by another application, consider making the external storage read-only in Nextcloud.

> [!NOTE]
> Removing an External Storage entry from the Nextcloud administration interface removes the mount configuration from Nextcloud. It does not delete the underlying host directory.

### Why not mount `/mnt`?

Avoid using:

```yaml
NEXTCLOUD_MOUNT: /mnt/
```

unless the entire host `/mnt` tree genuinely needs to be exposed to Nextcloud.

`NEXTCLOUD_MOUNT` is not required for:

- `NEXTCLOUD_DATADIR`
- local AIO Borg backups
- mounting backup disks on the host

Using a dedicated directory such as:

```text
/srv/server/media
```

provides a much narrower and more predictable mount scope.

> [!WARNING]
> Avoid exposing backup repositories or other unrelated host storage to the Nextcloud container.

---

## AIO Management Interface

With:

```yaml
ports:
  - 8081:8080
```

the AIO management interface is available at:

```text
https://<server-ip>:8081
```

Use the server's IP address when accessing the AIO management interface.

> [!NOTE]
> The AIO management interface is separate from the regular Nextcloud web interface.

---

## Applying AIO Configuration Changes

Changes to the mastercontainer configuration, such as adding or modifying environment variables, require the mastercontainer to be recreated.

For example, after adding:

```yaml
NEXTCLOUD_MOUNT: /srv/server/media
```

recreate the `nextcloud-aio-mastercontainer` using the updated Docker Compose configuration.

With Docker Compose, this can be done by applying the updated stack:

```bash
docker compose up -d
```

Docker Compose will recreate the container if its configuration has changed while preserving the `nextcloud_aio_mastercontainer` volume.

> [!IMPORTANT]
> Do not remove the `nextcloud_aio_mastercontainer` volume. It contains the persistent AIO configuration.

After the mastercontainer has been recreated:

1. Open the AIO management interface.
2. Select **Stop containers**.
3. Wait for all managed containers to stop.
4. Start them again using **Start containers** or **Start and update containers**.

This allows AIO to recreate managed containers with the updated configuration.

Afterwards, verify the resulting mount:

```bash
docker inspect nextcloud-aio-nextcloud \
  --format '{{range .Mounts}}{{println .Source "->" .Destination "RW=" .RW}}{{end}}'
```

Expected entries include:

```text
/srv/server/services/nextcloud/data -> /mnt/ncdata RW= true
/srv/server/media -> /srv/server/media RW= true
```

> [!NOTE]
> A regular host or container restart does not apply changes to the container configuration. Configuration changes that affect environment variables, mounts, devices, or other container settings require the affected container to be recreated.

---

## Built-in Borg Backup

Use the native AIO backup mechanism for Nextcloud instead of relying solely on a filesystem-level backup.

Example local backup location:

```text
/mnt/backup/nextcloud_aio
```

Do not append `/borg` manually. AIO creates its Borg repository inside the selected directory:

```text
/mnt/backup/nextcloud_aio/borg
```

Leave the remote Borg repository field empty when using local storage.

> [!IMPORTANT]
> Store the Borg encryption password or recovery information outside the server, preferably in a password manager.

### Retention

The Compose configuration uses:

```text
--keep-within=7d --keep-weekly=4 --keep-monthly=6
```

This retains:

- backups from the last 7 days
- 4 weekly backups
- 6 monthly backups

A successful backup can be verified in the AIO interface under **Backup and restore**.

### Verify the Backup Filesystem

Before running a local backup, verify that the expected backup filesystem is mounted:

```bash
findmnt -T /mnt/backup/nextcloud_aio
df -hT /mnt/backup/nextcloud_aio
```

> [!WARNING]
> Do not run a local backup if the expected external backup disk is not mounted. Otherwise, data may be written into the mountpoint directory on the root filesystem instead.

---

## Backup Disk Layout

A clean layout when sharing an external backup disk with another backup system such as Backrest:

```text
/mnt/backup/
├── backrest/
│   └── server/
├── nextcloud_aio/
│   └── borg/
└── lost+found/
```

The backup disk should be mounted before starting a local AIO backup.

For persistent host mounts, prefer a filesystem UUID in `/etc/fstab` rather than a device name such as `/dev/sda1`.

Example:

```text
UUID=<filesystem-uuid> /mnt/backup ext4 defaults,nofail,noatime,x-systemd.device-timeout=10s 0 2
```

The mount can be verified with:

```bash
findmnt /mnt/backup
systemctl status mnt-backup.mount
```

---

## Interaction with Backrest

A host-level Backrest plan may include:

```text
/srv/server/services/nextcloud/data
```

as part of a broader filesystem snapshot.

However, the **AIO Borg backup should be treated as the authoritative Nextcloud backup**, because it is designed specifically for the AIO deployment and handles the application state and database consistently.

If `/var/lib/docker` is excluded from the host-level backup, AIO-managed Docker volumes are not duplicated there.

A typical backup architecture may therefore look like:

```text
Host filesystem
├── system files
├── service configuration
├── /srv/server/services/
│   └── nextcloud/
│       └── data/
│
└── Backrest
    └── filesystem-level backup

Nextcloud AIO
└── Borg
    └── application-aware Nextcloud backup
```

> [!NOTE]
> Local External Storage exposed through `NEXTCLOUD_MOUNT` should be considered separately when designing the backup policy. Do not assume that files outside the main Nextcloud data directory are covered by the same backup mechanism.