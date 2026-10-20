---
title: Jellyseerr (deprecated, use Seerr)
description: Jellyseerr media request manager. Deprecated: new installs should use Seerr.
published: true
date: 2026-10-18T03:28:10.701Z
tags: docker, docker-compose, homelab, media, requests, deprecated
editor: markdown
dateCreated: 2026-09-08T21:11:08.018Z
---

# Jellyseerr (deprecated, use Seerr)

> **Deprecated.** This project has merged into [Seerr](/seerr). Use Seerr for new installs.
{.is-warning}

> Jellyseerr media request manager. Deprecated: new installs should use Seerr.
{.is-info}

_Last updated: Oct 2026_

## Overview

Jellyseerr is the Jellyfin-friendly fork of Overseerr. The project has merged into **[Seerr](/seerr)**, which is now the recommended request manager. This page is kept for existing installs and migration.

**Why you'd want it:** Only if you already run it. New installs should go straight to Seerr.

| Item | Value |
|---|---|
| Stack folder | `/opt/stacks/jellyseerr` |
| Category | Media |
| Main image | `fallenbagel/jellyseerr:latest` |
| Services | 1 |
| Web UI | http://<SERVER_IP>:5055 |
| Official docs | https://docs.seerr.dev/ |

## Requirements

- A Linux host with Docker Engine and the Docker Compose plugin (see [Docker](/docker))
- The shared folder layout and `.env`/`.secret` files from [Docker Folders](/dockerfolders) and [Environment](/environment)
- A Plex or Jellyfin server

## Compose

Folder layout:

```text
/opt/stacks/
├── _shared/
│   ├── .env        # shared, non-secret settings
│   └── .secret     # passwords, tokens and keys (chmod 600, never commit)
└── jellyseerr/
    └── compose.yaml
/opt/appdata/jellyseerr/   # persistent application data
```

`/opt/stacks/jellyseerr/compose.yaml`:

```yaml
---
services:
  jellyseerr:
    image: fallenbagel/jellyseerr:latest
    container_name: jellyseerr
    hostname: jellyseerr
    env_file:
      - /opt/stacks/_shared/.env
      - /opt/stacks/_shared/.secret
    logging:
      driver: json-file
      options:
        max-file: ${DOCKERLOGGING_MAXFILE:-10}  # Max number of log files
        max-size: ${DOCKERLOGGING_MAXSIZE:-200k}  # Max file size
    ports:
      - 5055:5055 # Jellyseer Web UI
    volumes:
      - /opt/appdata/jellyseerr/config:/app/config # Change to your desired config path
    network_mode: host
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
| 5055 | 5055 | tcp | `jellyseerr` | Jellyseer Web UI |
| host network | - | - | `jellyseerr` | 5055/tcp (web UI) |

## Volumes

| Host path | Container path | Service | Purpose |
|---|---|---|---|
| `/opt/appdata/jellyseerr/config` | `/app/config` | `jellyseerr` | Change to your desired config path |

## Resource limits

No limits are set. This app is light, so add `deploy.resources.limits` only if you need to cap it.

## Setup

1. Create the folders:
   ```bash
   sudo mkdir -p /opt/stacks/jellyseerr /opt/appdata/jellyseerr
   sudo chown -R $USER: /opt/stacks/jellyseerr
   ```
2. Add the variables above to `/opt/stacks/_shared/.env` and `/opt/stacks/_shared/.secret` (`chmod 600 .secret`).
3. Save the compose file as `/opt/stacks/jellyseerr/compose.yaml`.
4. Check the configuration, then start the stack:
   ```bash
   cd /opt/stacks/jellyseerr
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret config --quiet
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
   ```
5. Watch the first start: `docker compose logs -f`.
6. To migrate: stop Jellyseerr (`docker compose down`), copy `/opt/appdata/jellyseerr/config` to `/opt/appdata/seerr/config`, then start [Seerr](/seerr).

> Tip: create a small wrapper such as `alias dc='docker compose --env-file /opt/stacks/_shared/.env --env-file /opt/stacks/_shared/.secret'` so every stack uses the same env files.
{.is-success}

## Updating

```bash
cd /opt/stacks/jellyseerr
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret pull
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
docker image prune -f
```

Back up `/opt/appdata/jellyseerr` before major upgrades. [Watchtower](/watchtower) can automate updates for stateless apps.

## Troubleshooting

- Check status and logs: `docker compose ps` and `docker compose logs --tail=100`.
- **Port 5055 conflict with Seerr:** run only one of them.
- **A variable is empty or reported missing (`:?err`):** run the `config` command above to see which variable is not set in your env files.

## Related

- [Docker Stack Catalog](/docker-stacks)
- [Seerr](/seerr)

---
*Educational example. Review every path, port and credential before you deploy, and never publish your `.secret` file.*
