---
title: Directory Lister
description: Browsable web index of a shared folder with Directory Lister and Docker Compose.
published: true
date: 2026-10-18T03:27:49.685Z
tags: docker, docker-compose, documents, files, homelab, sharing, web-index
editor: markdown
dateCreated: 2026-09-08T21:26:12.838Z
---

# Directory Lister

> Browsable web index of a shared folder with Directory Lister and Docker Compose.
{.is-info}

_Last updated: Oct 2026_

## Overview

Directory Lister turns a folder into a clean, searchable web page with file icons, README rendering and zip downloads. Use it to share ISOs, drivers or software with your household.

**Why you'd want it:** A read-only download page that is easier than SMB shares for guests and phones.

| Item | Value |
|---|---|
| Stack folder | `/opt/stacks/directorylister` |
| Category | Documents |
| Main image | `directorylister/directorylister:latest` |
| Services | 1 |
| Web UI | http://<SERVER_IP>:30005 |
| Official docs | https://docs.directorylister.com/ |

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
└── directorylister/
    └── compose.yaml
/opt/appdata/directorylister/   # persistent application data
```

`/opt/stacks/directorylister/compose.yaml`:

```yaml
---
services:
  directory-lister:
    image: directorylister/directorylister:latest
    container_name: directorylister
    hostname: directorylister
    env_file:
      - /opt/stacks/_shared/.env
      - /opt/stacks/_shared/.secret
    environment:
      - APP_LANGUAGE=en
      - DISPLAY_READMES=true
      - READMES_FIRST=true
      - ZIP_DOWNLOADS=true
      - SORT_ORDER=type
      - REVERSE_SORT=false
      - SITE_TITLE=share
      - EXCLUDE_FILES=@eaDir
    logging:
      driver: json-file
      options:
        max-file: ${DOCKERLOGGING_MAXFILE:-10}  # Max number of log files
        max-size: ${DOCKERLOGGING_MAXSIZE:-200k}  # Max file size
      # Refer to Directory Lister's configuration documentation for more environment variables.
    ports:
      - 30005:80 # Replace <host_port> with the port you want to expose Directory Lister on your host
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /srv/share:/data
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
| 30005 | 80 | tcp | `directory-lister` | Replace <host_port> with the port you want to expose Directory Lister on your host |

## Volumes

| Host path | Container path | Service | Purpose |
|---|---|---|---|
| `/etc/localtime` | `/etc/localtime` | `directory-lister` | Host time zone (read-only) |
| `/srv/share` | `/data` | `directory-lister` | Application data |

## Resource limits

No limits are set. This app is light, so add `deploy.resources.limits` only if you need to cap it.

## Setup

1. Create the folders:
   ```bash
   sudo mkdir -p /opt/stacks/directorylister /opt/appdata/directorylister
   sudo chown -R $USER: /opt/stacks/directorylister
   ```
2. Add the variables above to `/opt/stacks/_shared/.env` and `/opt/stacks/_shared/.secret` (`chmod 600 .secret`).
3. Save the compose file as `/opt/stacks/directorylister/compose.yaml`.
4. Check the configuration, then start the stack:
   ```bash
   cd /opt/stacks/directorylister
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret config --quiet
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
   ```
5. Watch the first start: `docker compose logs -f`.
6. Put files in `/srv/share` and open `http://<SERVER_IP>:30005`.
7. Add a `README.md` to any folder to show it above the listing.

> Tip: create a small wrapper such as `alias dc='docker compose --env-file /opt/stacks/_shared/.env --env-file /opt/stacks/_shared/.secret'` so every stack uses the same env files.
{.is-success}

## Updating

```bash
cd /opt/stacks/directorylister
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret pull
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
docker image prune -f
```

Back up `/opt/appdata/directorylister` before major upgrades. [Watchtower](/watchtower) can automate updates for stateless apps.

## Troubleshooting

- Check status and logs: `docker compose ps` and `docker compose logs --tail=100`.
- **Synology `@eaDir` folders visible:** keep `EXCLUDE_FILES=@eaDir`.
- **Files missing:** the container needs read permission on the folder.
- Mount the share read-only (`:ro`) if nobody should change files through it.
- **A variable is empty or reported missing (`:?err`):** run the `config` command above to see which variable is not set in your env files.

## Related

- [Docker Stack Catalog](/docker-stacks)
- [File Browser](/filebrowser)
- [Filestash](/filestash)

---
*Educational example. Review every path, port and credential before you deploy, and never publish your `.secret` file.*
