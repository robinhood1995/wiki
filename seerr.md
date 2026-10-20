---
title: Seerr
description: Seerr media request manager for Plex, Jellyfin and Emby with Docker Compose.
published: true
date: 2026-10-18T03:27:20.571Z
tags: docker, docker-compose, homelab, jellyfin, media, plex, requests
editor: markdown
dateCreated: 2026-09-08T21:26:12.838Z
---

# Seerr

> Seerr media request manager for Plex, Jellyfin and Emby with Docker Compose.
{.is-info}

_Last updated: Oct 2026_

## Overview

Seerr is a request portal for your media server. Family and friends browse movies and shows, click Request, and Seerr hands the request to Sonarr or Radarr. It is the merged successor of Overseerr and Jellyseerr and supports Plex, Jellyfin and Emby.

**Why you'd want it:** One friendly place for requests instead of messages and spreadsheets, with per-user quotas and approval rules.

| Item | Value |
|---|---|
| Stack folder | `/opt/stacks/seerr` |
| Category | Media |
| Main image | `ghcr.io/seerr-team/seerr:latest` |
| Services | 1 |
| Web UI | http://<SERVER_IP>:5055 |
| Official docs | https://docs.seerr.dev/ |

## Requirements

- A Linux host with Docker Engine and the Docker Compose plugin (see [Docker](/docker))
- The shared folder layout and `.env`/`.secret` files from [Docker Folders](/dockerfolders) and [Environment](/environment)
- A Plex, Jellyfin or Emby server
- Sonarr and Radarr (see [Arr Stack](/arr-stack)) for automatic downloads

## Compose

Folder layout:

```text
/opt/stacks/
├── _shared/
│   ├── .env        # shared, non-secret settings
│   └── .secret     # passwords, tokens and keys (chmod 600, never commit)
└── seerr/
    └── compose.yaml
/opt/appdata/seerr/   # persistent application data
```

`/opt/stacks/seerr/compose.yaml`:

```yaml
---
services:
  seerr:
    image: ghcr.io/seerr-team/seerr:latest
    init: true
    container_name: seerr
    hostname: seerr
    user: "${PUID}:${PGID}"
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
      - /opt/appdata/seerr/config:/app/config # Change to your desired config path
    network_mode: host
    healthcheck:
      test: wget --no-verbose --tries=1 --spider http://localhost:5055/api/v1/status || exit 1
      start_period: 20s
      timeout: 3s
      interval: 15s
      retries: 3
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
| 5055 | 5055 | tcp | `seerr` | Jellyseer Web UI |
| host network | - | - | `seerr` | 5055/tcp (web UI) |

## Volumes

| Host path | Container path | Service | Purpose |
|---|---|---|---|
| `/opt/appdata/seerr/config` | `/app/config` | `seerr` | Change to your desired config path |

## Resource limits

No limits are set. This app is light, so add `deploy.resources.limits` only if you need to cap it.

## Setup

1. Create the folders:
   ```bash
   sudo mkdir -p /opt/stacks/seerr /opt/appdata/seerr
   sudo chown -R $USER: /opt/stacks/seerr
   ```
2. Add the variables above to `/opt/stacks/_shared/.env` and `/opt/stacks/_shared/.secret` (`chmod 600 .secret`).
3. Save the compose file as `/opt/stacks/seerr/compose.yaml`.
4. Check the configuration, then start the stack:
   ```bash
   cd /opt/stacks/seerr
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret config --quiet
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
   ```
5. Watch the first start: `docker compose logs -f`.
6. Open `http://<SERVER_IP>:5055` and sign in with your media server account.
7. Add your media server and run a library scan.
8. Under **Settings > Services**, add Sonarr and Radarr with their API keys.
9. Set default permissions and request limits for new users.

> Tip: create a small wrapper such as `alias dc='docker compose --env-file /opt/stacks/_shared/.env --env-file /opt/stacks/_shared/.secret'` so every stack uses the same env files.
{.is-success}

## Updating

```bash
cd /opt/stacks/seerr
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret pull
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
docker image prune -f
```

Back up `/opt/appdata/seerr` before major upgrades. [Watchtower](/watchtower) can automate updates for stateless apps.

## Troubleshooting

- Check status and logs: `docker compose ps` and `docker compose logs --tail=100`.
- **Permission denied on /app/config:** the folder must be owned by `PUID:PGID` because the container runs as that user (`chown -R <PUID>:<PGID> /opt/appdata/seerr`).
- **Port 5055 already in use:** Jellyseerr and Seerr use the same port. Stop Jellyseerr before starting Seerr.
- **Migrating from Jellyseerr/Overseerr:** stop the old container and copy its config folder into `/opt/appdata/seerr/config` before the first start.
- **A variable is empty or reported missing (`:?err`):** run the `config` command above to see which variable is not set in your env files.

## Related

- [Docker Stack Catalog](/docker-stacks)
- [Jellyseerr](/jellyseerr)
- [Arr Stack](/arr-stack)
- [Plex](/plex)
- [Jellyfin](/jellyfin)

---
*Educational example. Review every path, port and credential before you deploy, and never publish your `.secret` file.*
