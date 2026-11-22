---
title: Backrest
description: Web UI for Restic backups with scheduled snapshots and restore, using Docker Compose.
published: true
date: 2026-11-22T06:33:47.543Z
tags: backup, docker, docker-compose, homelab, restic
editor: markdown
dateCreated: 2026-11-22T06:33:47.543Z
---

# Backrest

> Web UI for Restic backups (Backrest) with scheduled snapshots, restore and repository management, using Docker Compose.
{.is-info}

_Last updated: Nov 2026_

## Overview

Backrest is a web front end for Restic. It schedules backups, reports on snapshots and lets you restore files from a browser. Repositories can live on a local disk, a share, or an off-site backend that Restic supports.

**Why you'd want it:** Planned backups you can see and restore without remembering Restic commands.

| Item | Value |
|---|---|
| Stack folder | `/opt/stacks/backrest` |
| Category | Stack management |
| Main image | `garethgeorge/backrest:latest` |
| Services | 1 |
| Web UI | http://<SERVER_IP>:9898 |
| Official docs | https://github.com/garethgeorge/backrest |

## Requirements

- A Linux host with Docker Engine and the Docker Compose plugin (see [Docker](/docker))
- The shared folder layout and `.env`/`.secret` files from [Docker Folders](/dockerfolders) and [Environment](/environment)
- Disk or remote storage for the Restic repository (this example uses `/srv/backups`)
- A strong repository password stored only in `.secret` or in the Backrest UI, never on this page

## Compose

Folder layout:

```text
/opt/stacks/
├── _shared/
│   ├── .env        # shared, non-secret settings
│   └── .secret     # passwords, tokens and keys (chmod 600, never commit)
└── backrest/
    └── compose.yaml
/opt/appdata/backrest/   # persistent application data
```

`/opt/stacks/backrest/compose.yaml`:

```yaml
---
services:
  backrest:
    image: garethgeorge/backrest:latest
    container_name: backrest
    hostname: backrest
    env_file:
      - /opt/stacks/_shared/.env
      - /opt/stacks/_shared/.secret
    environment:
      - BACKREST_PORT=9898
      - BACKREST_DATA=/data
      - BACKREST_CONFIG=/config/config.json
      - XDG_CACHE_HOME=/cache
      - TZ=${TZ}
    logging:
      driver: json-file
      options:
        max-file: ${DOCKERLOGGING_MAXFILE:-10}
        max-size: ${DOCKERLOGGING_MAXSIZE:-200k}
    ports:
      - 9898:9898
    volumes:
      - /opt/appdata/backrest/data:/data
      - /opt/appdata/backrest/config:/config
      - /opt/appdata/backrest/cache:/cache
      - /opt/appdata:/userdata:ro
      - /srv/backups:/repos
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
| PUID | .env | User ID the container runs as (see Docker Folders) | 1000 |
| PGID | .env | Group ID the container runs as | 1000 |
| TZ | .env | Time zone in tz database format, e.g. America/New_York | <TZ> |
| DOCKERLOGGING_MAXFILE | .env | Number of rotated container log files to keep | 10 |
| DOCKERLOGGING_MAXSIZE | .env | Maximum size of each container log file | 200k |

Set the Restic repository password in the Backrest UI on first use (or in the config it writes under `/opt/appdata/backrest/config`). Do not put live passwords in this wiki page.

## Ports

| Host port | Container port | Protocol | Service | Purpose |
|---|---|---|---|---|
| 9898 | 9898 | tcp | backrest | Web UI |

## Volumes

| Host path | Container path | Service | Purpose |
|---|---|---|---|
| `/opt/appdata/backrest/data` | `/data` | backrest | Application data |
| `/opt/appdata/backrest/config` | `/config` | backrest | Backrest config |
| `/opt/appdata/backrest/cache` | `/cache` | backrest | Restic cache |
| `/opt/appdata` | `/userdata` | backrest | Example backup source (read-only) |
| `/srv/backups` | `/repos` | backrest | Local repository path |

## Resource limits

No limits are set. Backups can be I/O heavy; add `deploy.resources.limits` if the host is a small machine.

## Setup

1. Create the folders:
   ```bash
   sudo mkdir -p /opt/stacks/backrest /opt/appdata/backrest
   sudo chown -R $USER: /opt/stacks/backrest
   ```
2. Add the variables above to `/opt/stacks/_shared/.env` and `/opt/stacks/_shared/.secret` (`chmod 600 .secret`).
3. Save the compose file as `/opt/stacks/backrest/compose.yaml`.
4. Check the configuration, then start the stack:
   ```bash
   cd /opt/stacks/backrest
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret config --quiet
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
   ```
5. Watch the first start: `docker compose logs -f`.

6. Open `http://<SERVER_IP>:9898` and create the first admin user.
7. Add a repository under `/repos` (or a remote backend), then add the paths you want to back up (for example `/userdata`).
8. Create a schedule and run one backup by hand to confirm it finishes.

> Tip: create a small wrapper such as `alias dc='docker compose --env-file /opt/stacks/_shared/.env --env-file /opt/stacks/_shared/.secret'` so every stack uses the same env files.
{.is-success}
## Updating

```bash
cd /opt/stacks/backrest
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret pull
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
docker image prune -f
```

Back up `/opt/appdata/backrest` before major upgrades. [Watchtower](/watchtower) can automate updates for stateless apps.

## Troubleshooting

- Check status and logs: `docker compose ps` and `docker compose logs --tail=100`.
- **Permission denied on `/userdata` or `/repos`:** the container user must be able to read sources and write the repository. Match ownership to `PUID`/`PGID` where you can, or adjust the bind mounts.
- **Do not expose port 9898 to the internet** without a login-aware reverse proxy. See [oauth2-proxy](/oauth2-proxy) if the app is reachable beyond your LAN.
- **A variable is empty or reported missing (`:?err`):** run the `config` command above to see which variable is not set in your env files.

## Related

- [Docker Stack Catalog](/docker-stacks)
- [Watchtower](/watchtower)
- [oauth2-proxy](/oauth2-proxy)

---
*Educational example. Review every path, port and credential before you deploy, and never publish your `.secret` file.*
