---
title: Dispatcharr
description: IPTV / M3U stream and EPG manager with Dispatcharr, PostgreSQL and NVIDIA GPU, using Docker Compose.
published: true
date: 2026-10-18T03:27:52.316Z
tags: docker, docker-compose, homelab, media, epg, gpu, iptv, m3u
editor: markdown
dateCreated: 2026-09-08T21:26:12.838Z
---

# Dispatcharr

> IPTV / M3U stream and EPG manager with Dispatcharr, PostgreSQL and NVIDIA GPU, using Docker Compose.
{.is-info}

_Last updated: Oct 2026_

## Overview

Dispatcharr manages IPTV playlists (M3U) and guide data (EPG). It merges sources, filters and renumbers channels, and serves a clean playlist and HDHomeRun-style tuner to Plex, Jellyfin or Emby. A GPU can be used for stream transcoding.

**Why you'd want it:** Tame large IPTV playlists into a tidy channel lineup for your media server.

| Item | Value |
|---|---|
| Stack folder | `/opt/stacks/dispatcharr` |
| Category | Media |
| Main image | `ghcr.io/dispatcharr/dispatcharr:latest` |
| Services | 1 |
| Web UI | http://<SERVER_IP>:9191 |
| Official docs | https://github.com/Dispatcharr/Dispatcharr |

## Requirements

- A Linux host with Docker Engine and the Docker Compose plugin (see [Docker](/docker))
- The shared folder layout and `.env`/`.secret` files from [Docker Folders](/dockerfolders) and [Environment](/environment)
- PostgreSQL and Redis (external, set `DATABASE_URL` and `REDIS_URL`)
- Optional: NVIDIA GPU with the NVIDIA Container Toolkit

## Compose

Folder layout:

```text
/opt/stacks/
├── _shared/
│   ├── .env        # shared, non-secret settings
│   └── .secret     # passwords, tokens and keys (chmod 600, never commit)
└── dispatcharr/
    └── compose.yaml
/opt/appdata/dispatcharr/   # persistent application data
```

`/opt/stacks/dispatcharr/compose.yaml`:

```yaml
---
services:
  dispatcharr:
    image: ghcr.io/dispatcharr/dispatcharr:latest
    container_name: dispatcharr
    hostname: dispatcharr
    env_file:
      - /opt/stacks/_shared/.env
      - /opt/stacks/_shared/.secret
    environment:
      - DISPATCHARR_LOG_LEVEL=error
      - DISPATCHARR_ENV=production
      - DATABASE_URL=postgresql://<DB_USER>:<DB_PASSWORD>@<DB_HOST>:5432/dispatcharr
      # REDIS_URL comes from _shared/.env via env_file
      # NVIDIA GPU Passthrough
      - NVIDIA_VISIBLE_DEVICES=all
      - NVIDIA_DRIVER_CAPABILITIES=all
    logging:
      driver: json-file
      options:
        max-file: ${DOCKERLOGGING_MAXFILE:-10}  # Max number of log files
        max-size: ${DOCKERLOGGING_MAXSIZE:-200k}  # Max file size
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 2000M
        reservations:
          devices:
            - driver: nvidia
              count: 1 # Use 'all' if you have multiple GPUs
              capabilities: [gpu, video]
    ports:
      - 9191:9191
    volumes:
      - /opt/appdata/dispatcharr/data:/data
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
REDIS_URL=redis://<REDIS_HOST>:6379/0
```

| Variable | File | Purpose | Example |
|---|---|---|---|
| `PUID` | `.env` | User ID the container runs as (see Docker Folders) | `1000` |
| `PGID` | `.env` | Group ID the container runs as | `1000` |
| `TZ` | `.env` | Time zone in tz database format, e.g. America/New_York | `<TZ>` |
| `DOCKERLOGGING_MAXFILE` | `.env` | Number of rotated container log files to keep | `10` |
| `DOCKERLOGGING_MAXSIZE` | `.env` | Maximum size of each container log file | `200k` |
| `REDIS_URL` | `.env` | Redis connection string for Dispatcharr | `redis://<REDIS_HOST>:6379/0` |

## Ports

| Host port | Container port | Protocol | Service | Purpose |
|---|---|---|---|---|
| 9191 | 9191 | tcp | `dispatcharr` | Web UI |

## Volumes

| Host path | Container path | Service | Purpose |
|---|---|---|---|
| `/opt/appdata/dispatcharr/data` | `/data` | `dispatcharr` | Application data |

## Resource limits

Recommended limits, already set in the compose file under `deploy.resources`. Raise them if the app is starved, or lower them on small hosts.

| Service | CPU limit | Memory limit | Memory reservation | GPU |
|---|---|---|---|---|
| `dispatcharr` | 2.0 | 2000M | - | nvidia x 1 (gpu, video) |

## Setup

1. Create the folders:
   ```bash
   sudo mkdir -p /opt/stacks/dispatcharr /opt/appdata/dispatcharr
   sudo chown -R $USER: /opt/stacks/dispatcharr
   ```
2. Add the variables above to `/opt/stacks/_shared/.env` and `/opt/stacks/_shared/.secret` (`chmod 600 .secret`).
3. Save the compose file as `/opt/stacks/dispatcharr/compose.yaml`.
4. Check the configuration, then start the stack:
   ```bash
   cd /opt/stacks/dispatcharr
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret config --quiet
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
   ```
5. Watch the first start: `docker compose logs -f`.
6. Create a `dispatcharr` database and user in PostgreSQL.
7. Open `http://<SERVER_IP>:9191` and create the admin user.
8. Add your M3U and EPG sources, then build channel groups.
9. Add the Dispatcharr tuner or M3U URL to Plex or Jellyfin Live TV.

> Tip: create a small wrapper such as `alias dc='docker compose --env-file /opt/stacks/_shared/.env --env-file /opt/stacks/_shared/.secret'` so every stack uses the same env files.
{.is-success}

## Updating

```bash
cd /opt/stacks/dispatcharr
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret pull
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
docker image prune -f
```

Back up `/opt/appdata/dispatcharr` before major upgrades. [Watchtower](/watchtower) can automate updates for stateless apps.

## Troubleshooting

- Check status and logs: `docker compose ps` and `docker compose logs --tail=100`.
- **GPU not used:** check `nvidia-smi` on the host and that the NVIDIA runtime is installed.
- **Container exits on start:** usually `DATABASE_URL` or `REDIS_URL` cannot be reached.
- No GPU? Remove the `reservations.devices` block and the NVIDIA variables.
- **A variable is empty or reported missing (`:?err`):** run the `config` command above to see which variable is not set in your env files.

## Related

- [Docker Stack Catalog](/docker-stacks)
- [Plex](/plex)
- [Jellyfin](/jellyfin)

---
*Educational example. Review every path, port and credential before you deploy, and never publish your `.secret` file.*
