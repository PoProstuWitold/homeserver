# Backrest

[Backrest](https://github.com/garethgeorge/backrest) is a web UI and orchestrator for [restic](https://restic.net/).

This setup backs up the host filesystem to a separate external disk. It is intended for a Docker host where persistent service data is stored under `/srv/server/services`.

> [!WARNING]
> Read the entire guide before executing any commands or making configuration changes. Some steps depend on decisions and settings described later in the guide.

> [!IMPORTANT]
> The backup repository should live on a filesystem separate from the system disk.
>
> The examples below use `/mnt/backup` as the mount point for an external backup disk.

---

## External Backup Disk Setup

### Identify the Disk

Connect the backup disk and inspect available block devices:

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,LABEL,UUID,MOUNTPOINTS,MODEL
```

Optionally inspect the disk health before using it:

```bash
sudo smartctl -a /dev/sdX
```

Replace `/dev/sdX` with the actual device.

> [!CAUTION]
> Always verify the device name before partitioning or formatting a disk.
>
> The commands in the next section destroy existing partition and filesystem data on the selected device.

### Partition and Format a New Disk

For a new or disposable disk, create a GPT partition table and a single ext4 partition:

```bash
sudo parted /dev/sdX --script \
  mklabel gpt \
  mkpart primary ext4 0% 100%
```

Format the resulting partition:

```bash
sudo mkfs.ext4 -L BACKUP /dev/sdX1
```

Verify the filesystem:

```bash
lsblk -f /dev/sdX
sudo blkid /dev/sdX1
```

> [!NOTE]
> Skip the partitioning and formatting steps when using an existing filesystem that already contains data.

### Create the Mount Point

```bash
sudo mkdir -p /mnt/backup
```

Obtain the filesystem UUID:

```bash
sudo blkid /dev/sdX1
```

Example output:

```text
/dev/sdX1: LABEL="BACKUP" UUID="<filesystem-uuid>" TYPE="ext4"
```

Use the UUID rather than `/dev/sdX1` in `/etc/fstab`. Device names such as `/dev/sda` may change between boots.

Add:

```text
UUID=<filesystem-uuid> /mnt/backup ext4 defaults,nofail,noatime,x-systemd.device-timeout=10s 0 2
```

to:

```text
/etc/fstab
```

The options used here provide:

- `defaults` - standard filesystem mount options
- `nofail` - allow the system to boot even if the backup disk is unavailable
- `noatime` - avoid updating file access timestamps
- `x-systemd.device-timeout=10s` - limit how long systemd waits for the device during boot

Reload systemd configuration:

```bash
sudo systemctl daemon-reload
```

Mount filesystems defined in `/etc/fstab`:

```bash
sudo mount -a
```

### Verify the Mount

Validate the complete `fstab` configuration:

```bash
sudo findmnt --verify --verbose
```

Verify the backup filesystem:

```bash
findmnt /mnt/backup
df -hT /mnt/backup
```

The result should show the external filesystem mounted at:

```text
/mnt/backup
```

The corresponding systemd mount unit can also be inspected:

```bash
systemctl status mnt-backup.mount --no-pager
```

> [!TIP]
> After configuring the disk, reboot the host once and repeat the checks above. This verifies that the filesystem is mounted automatically during a real boot.

### Create Backup Directories

Create separate directories for each backup system:

```bash
sudo mkdir -p /mnt/backup/backrest
sudo mkdir -p /mnt/backup/nextcloud_aio
```

A simple layout is:

```text
/mnt/backup/
├── backrest/
├── nextcloud_aio/
└── lost+found/
```

> [!WARNING]
> Verify that `/mnt/backup` is actually mounted before creating repository directories.
>
> Otherwise, the directories may accidentally be created on the system disk underneath the intended mount point.

A useful check is:

```bash
findmnt -T /mnt/backup/backrest
```

It should resolve to the external backup filesystem.

---

## Docker Compose

`docker-compose.yml`

```yaml
services:
  backrest:
    image: ghcr.io/garethgeorge/backrest:latest
    container_name: backrest
    hostname: backrest
    volumes:
      - type: bind
        source: /srv/server/services/backrest/data
        target: /data
        bind:
          create_host_path: false
      - type: bind
        source: /srv/server/services/backrest/config
        target: /config
        bind:
          create_host_path: false
      - type: bind
        source: /srv/server/services/backrest/cache
        target: /cache
        bind:
          create_host_path: false
      - type: bind
        source: /srv/server/services/backrest/tmp
        target: /tmp
        bind:
          create_host_path: false
      # Backup Source
      - type: bind
        source: /
        target: /host
        read_only: true
        bind:
          propagation: rslave
          create_host_path: false
      # External SSD
      - type: bind
        source: /mnt/backup/backrest
        target: /repos
        bind:
          create_host_path: false
    environment:
      - BACKREST_DATA=/data
      - BACKREST_CONFIG=/config/config.json
      - XDG_CACHE_HOME=/cache
      - TMPDIR=/tmp
      - TZ=Europe/Warsaw
    security_opt:
      - label=disable
      - no-new-privileges=true
    ports:
      - "9898:9898"
    restart: unless-stopped
```

> [!IMPORTANT]
> `create_host_path: false` is intentional.
>
> If `/mnt/backup/backrest` is unavailable, Docker should fail to create the container instead of silently creating the repository directory on the system disk.

The host filesystem is exposed to Backrest as:

```text
/ -> /host
```

in read-only mode.

The external repository directory is exposed as:

```text
/mnt/backup/backrest -> /repos
```

and remains writable so restic can create and maintain repositories.

### Verify the Mounts

After starting Backrest:

```bash
docker inspect backrest \
  --format '{{range .Mounts}}{{println .Source "->" .Destination "RW=" .RW}}{{end}}'
```

The relevant entries should include:

```text
/ -> /host RW= false
/mnt/backup/backrest -> /repos RW= true
```

Verify that `/repos` is actually backed by the external filesystem:

```bash
docker exec backrest df -hT /repos
```

> [!WARNING]
> Do not start a backup until `/repos` has been verified to reside on the intended external filesystem.

---

## Repository

Create a local repository, for example:

```text
Repository name:
server

Repository URI:
/repos/server
```

On the host this becomes:

```text
/mnt/backup/backrest/server
```

Recommended repository settings:

```text
Auto Unlock: ON
Shared: OFF
I/O Priority: IO_BEST_EFFORT_LOW
CPU Priority: CPU_DEFAULT
```

### Repository Maintenance

Suggested maintenance schedule:

```text
Prune:
0 4 1 * *
Local time
Max unused after prune: 10%

Check:
0 3 15 * *
Local time
Read Data: 100%

Repo-wide Forget Policy:
Disabled
```

Retention should be configured per backup plan instead of globally.

> [!IMPORTANT]
> Store the restic repository password outside the server, preferably in a password manager.
>
> Losing the repository password means losing access to the encrypted backups.

---

## Backup Plan

Example plan name:

```text
system
```

Suggested schedule:

```text
0 2 * * *
```

This runs the backup every day at `02:00` local time.

### Paths

When using `--one-file-system`, explicitly add filesystems that should be included:

```text
/host
/host/boot
/host/boot/efi
```

Adjust these paths to match the host's actual mount layout.

For example, inspect it with:

```bash
findmnt
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINTS
```

### Excludes

Example excludes:

```text
/host/proc
/host/sys
/host/dev
/host/run
/host/tmp
/host/mnt/backup
/host/var/tmp
/host/var/cache
/host/var/lib/docker
/host/srv/server/media
/host/srv/server/services/backrest/cache
/host/srv/server/services/backrest/tmp
```

The exclusions serve different purposes:

- `/proc`, `/sys`, `/dev`, `/run` - runtime and pseudo filesystems
- `/tmp`, `/var/tmp`, `/var/cache` - temporary or regenerable data
- `/mnt/backup` - prevents recursively backing up the backup disk
- `/var/lib/docker` - Docker images, layers, cache and named volumes
- `/srv/server/media` - optional exclusion for large or reproducible media
- Backrest cache and temporary directories - regenerable Backrest state

> [!IMPORTANT]
> Excluding `/var/lib/docker` also excludes Docker named volumes stored there.
>
> Important named volumes must therefore be backed up separately or replaced with bind-mounted persistent data where appropriate.

### Backup Flags

```text
--one-file-system
```

This prevents restic from crossing into other mounted filesystems while processing a backup path.

For example, backing up:

```text
/host
```

will not automatically cross into separate `/boot` and `/boot/efi` filesystems.

That is why they are explicitly included:

```text
/host
/host/boot
/host/boot/efi
```

> [!NOTE]
> `/host/mnt/backup` should still remain explicitly excluded as an additional safeguard against backup recursion.

### Retention

A moderate retention policy:

```text
Hourly:  0
Daily:   7
Weekly:  4
Monthly: 6
Yearly:  1
Latest:  3
```

This provides short-term rollback points while still preserving progressively older snapshots.

---

## Recovery Metadata

A filesystem backup contains the files required to rebuild much of the system, but it is not a raw block-level disk image.

For easier bare-metal recovery, keep a small metadata directory inside the backed-up filesystem:

```text
/srv/server/backup_metadata
```

Useful files include:

```text
packages.txt
repositories.txt
lsblk.txt
blkid.txt
pvs.txt
vgs.txt
lvs.txt
disk_partitions.sfdisk
os_release.txt
uname.txt
```

### Generate Recovery Metadata

```bash
sudo mkdir -p /srv/server/backup_metadata

sudo rpm -qa --qf '%{NAME}-%{VERSION}-%{RELEASE}.%{ARCH}\n' \
  | sort \
  | sudo tee /srv/server/backup_metadata/packages.txt > /dev/null

sudo dnf repolist --enabled \
  | sudo tee /srv/server/backup_metadata/repositories.txt > /dev/null

sudo lsblk -o NAME,SIZE,TYPE,FSTYPE,FSVER,LABEL,UUID,PARTUUID,MOUNTPOINTS,MODEL \
  | sudo tee /srv/server/backup_metadata/lsblk.txt > /dev/null

sudo blkid \
  | sudo tee /srv/server/backup_metadata/blkid.txt > /dev/null

sudo pvs | sudo tee /srv/server/backup_metadata/pvs.txt > /dev/null
sudo vgs | sudo tee /srv/server/backup_metadata/vgs.txt > /dev/null
sudo lvs -a -o +devices | sudo tee /srv/server/backup_metadata/lvs.txt > /dev/null

sudo sfdisk --dump /dev/nvme0n1 \
  | sudo tee /srv/server/backup_metadata/disk_partitions.sfdisk > /dev/null

sudo sh -c 'cat /etc/os-release > /srv/server/backup_metadata/os_release.txt'
uname -a | sudo tee /srv/server/backup_metadata/uname.txt > /dev/null
```

> [!NOTE]
> Adjust the disk device in the `sfdisk` command to match the actual system disk.

These files make it easier to reconstruct:

- installed packages
- enabled repositories
- filesystem UUIDs
- mount layout
- LVM layout
- disk partitioning
- operating system version
- kernel information

> [!TIP]
> Regenerate this metadata periodically or after meaningful storage, package, or system configuration changes.

---

## Backup Disk Layout

A clean layout for sharing one external disk between multiple independent backup systems:

```text
/mnt/backup/
├── backrest/
│   └── server/
├── nextcloud_aio/
│   └── borg/
└── lost+found/
```

This keeps the restic and Borg repositories separate while allowing them to share the same physical backup filesystem.

Verify the disk before backup operations:

```bash
findmnt /mnt/backup
df -hT /mnt/backup
```

Example expected relationship:

```text
External filesystem
└── /mnt/backup
    ├── backrest/
    │   └── server/
    └── nextcloud_aio/
        └── borg/
```

> [!WARNING]
> The external disk must be mounted before starting a backup.
>
> A directory named `/mnt/backup` existing on the host does not by itself prove that the external filesystem is mounted.

---

## Interaction with Nextcloud AIO

A host-level Backrest plan may include the Nextcloud data directory:

```text
/srv/server/services/nextcloud/data
```

as part of the broader filesystem backup.

However, the **native Nextcloud AIO Borg backup should be treated as the authoritative Nextcloud backup**, because it is designed to back up the AIO application and database consistently.

A typical arrangement is therefore:

```text
Backrest
└── Host filesystem
    ├── system configuration
    ├── user files
    ├── service configuration
    └── /srv/server/services/

Nextcloud AIO
└── Borg
    └── Nextcloud-specific backup
```

If `/var/lib/docker` is excluded from Backrest, AIO-managed Docker volumes are not duplicated in the host-level backup.

> [!NOTE]
> Data exposed to Nextcloud as Local External Storage should have its own explicit backup policy. Do not assume that external storage is covered by the native AIO Borg backup.

---

## Verification

After the initial backup, verify that the expected files are present in the snapshot and that excluded paths are absent.

Useful paths to check include:

```text
/host/etc
/host/etc/ssh
/host/home
/host/root
/host/boot
/host/boot/efi
/host/srv/server
/host/srv/server/backup_metadata
/host/srv/server/services
```

Expected exclusions may include:

```text
/host/mnt/backup
/host/var/lib/docker
/host/srv/server/media
```

> [!IMPORTANT]
> A backup should not be considered verified merely because the backup job completed successfully.
>
> Inspect snapshots and periodically perform restore tests.