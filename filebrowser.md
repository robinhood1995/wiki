---
title: File Browser
description: File Browser simple web file manager with sharing links, using Docker Compose.
published: true
date: 2026-10-20T20:01:18.188Z
tags: docker, docker-compose, documents, file-manager, files, homelab, sharing
editor: markdown
dateCreated: 2026-09-08T21:15:35.866Z
---

# File Browser

> File Browser simple web file manager with sharing links, using Docker Compose.
{.is-info}

_Last updated: Oct 2026_

## Overview

File Browser gives you a clean web interface for one folder on your server: upload, download, rename, move, preview and edit files, and create share links. Users and permissions are kept in a small database.

**Why you'd want it:** Easy file access from phones and guests without setting up SMB or SFTP.

| Item | Value |
|---|---|
| Stack folder | `/opt/stacks/filebrowser` |
| Category | Documents |
| Main image | `filebrowser/filebrowser:s6` |
| Services | 1 |
| Web UI | http://<SERVER_IP>:30004 |
| Official docs | https://filebrowser.org/ |

## Requirements

- A Linux host with Docker Engine and the Docker Compose plugin (see [Docker](/docker))
- The shared folder layout and `.env`/`.secret` files from [Docker Folders](/dockerfolders) and [Environment](/environment)
- A folder to publish, e.g. `/srv/share`

## Compose

Folder layout:

```text
/opt/stacks/
├── _shared/
│   ├── .env        # shared, non-secret settings
│   └── .secret     # passwords, tokens and keys (chmod 600, never commit)
└── filebrowser/
    └── compose.yaml
/opt/appdata/filebrowser/   # persistent application data
```

`/opt/stacks/filebrowser/compose.yaml`:

```yaml
---
services:
  filebrowser:
    image: filebrowser/filebrowser:s6
    container_name: filebrowser
    hostname: filebrowser
    env_file:
      - /opt/stacks/_shared/.env
      - /opt/stacks/_shared/.secret
    environment:
      - FB_NOAUTH=false  # true turns off the login page; only on a trusted LAN
    logging:
      driver: json-file
      options:
        max-file: ${DOCKERLOGGING_MAXFILE:-10}  # Max number of log files
        max-size: ${DOCKERLOGGING_MAXSIZE:-200k}  # Max file size
    volumes:
      - /srv/share:/srv  # the folder you want to browse
      - /opt/appdata/filebrowser/config:/config  # Persistent configuration directory
      - /opt/appdata/filebrowser/database:/database  # Persistent database directory
    ports:
      - 30004:80  # web UI on host port 30004
    restart: unless-stopped
```

## Environment variables

Compose reads these from the shared files with `--env-file`. Replace every `<PLACEHOLDER>` with your own value.

`/opt/stacks/_shared/.env`:

```ini
PUID=1000
PGID=1000
TZ=<TZ>
DOCKERLOGGING_MAXFILE=10
DOCKERLOGGING_MAXSIZE=200k
```

| Variable | File | Purpose | Example |
|---|---|---|---|
| `PUID` | `.env` | User ID the container runs as (see Docker Folders) | `1000` |
| `PGID` | `.env` | Group ID the container runs as | `1000` |
| `TZ` | `.env` | Time zone in tz database format, e.g. America/New_York | `<TZ>` |
| `DOCKERLOGGING_MAXFILE` | `.env` | Number of rotated container log files to keep | `10` |
| `DOCKERLOGGING_MAXSIZE` | `.env` | Maximum size of each container log file | `200k` |

## Ports

| Host port | Container port | Protocol | Service | Purpose |
|---|---|---|---|---|
| 30004 | 80 | tcp | `filebrowser` | web UI on host port 30004 |

## Volumes

| Host path | Container path | Service | Purpose |
|---|---|---|---|
| `/srv/share` | `/srv` | `filebrowser` | the folder you want to browse |
| `/opt/appdata/filebrowser/config` | `/config` | `filebrowser` | Persistent configuration directory |
| `/opt/appdata/filebrowser/database` | `/database` | `filebrowser` | Persistent database directory |

## Resource limits

No limits are set. This app is light, so add `deploy.resources.limits` only if you need to cap it.

## Setup

1. Create the folders:
   ```bash
   sudo mkdir -p /opt/stacks/filebrowser /opt/appdata/filebrowser
   sudo chown -R $USER: /opt/stacks/filebrowser
   ```
2. Add the variables above to `/opt/stacks/_shared/.env` and `/opt/stacks/_shared/.secret` (`chmod 600 .secret`).
3. Save the compose file as `/opt/stacks/filebrowser/compose.yaml`.
4. Check the configuration, then start the stack:
   ```bash
   cd /opt/stacks/filebrowser
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret config --quiet
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
   ```
5. Watch the first start: `docker compose logs -f`.
6. Start the stack, then read the first admin password from `docker logs filebrowser` (older images use `admin` / `admin`).
7. Open `http://<SERVER_IP>:30004`, sign in and change the password under **Settings > Profile**.
8. Create users for family members and limit each to a scope folder.

> Tip: create a small wrapper such as `alias dc='docker compose --env-file /opt/stacks/_shared/.env --env-file /opt/stacks/_shared/.secret'` so every stack uses the same env files.
{.is-success}

## Updating

```bash
cd /opt/stacks/filebrowser
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret pull
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
docker image prune -f
```

Back up `/opt/appdata/filebrowser` before major upgrades. [Watchtower](/watchtower) can automate updates for stateless apps.

## Troubleshooting

- Check status and logs: `docker compose ps` and `docker compose logs --tail=100`.
- **No login page:** `FB_NOAUTH=true` disables authentication. Keep it `false` unless the app is on a trusted LAN only.
- **Permission denied on upload:** the mounted folder must be writable by `PUID:PGID`.
- **Settings lost on update:** keep `/database` and `/config` on persistent storage.
- **A variable is empty or reported missing (`:?err`):** run the `config` command above to see which variable is not set in your env files.

## Related

- [Docker Stack Catalog](/docker-stacks)
- [Filestash](/filestash)
- [Directory Lister](/directorylister)
- [Double Commander](/doublecommander)

---
*Educational example. Review every path, port and credential before you deploy, and never publish your `.secret` file.*
