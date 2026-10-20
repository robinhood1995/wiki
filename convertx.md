---
title: ConvertX
description: Self-hosted file converter for documents, images, audio and video with ConvertX and Docker Compose.
published: true
date: 2026-10-18T03:27:44.486Z
tags: docker, docker-compose, documents, files, homelab, converter, ffmpeg
editor: markdown
dateCreated: 2026-09-08T21:26:12.838Z
---

# ConvertX

> Self-hosted file converter for documents, images, audio and video with ConvertX and Docker Compose.
{.is-info}

_Last updated: Oct 2026_

## Overview

ConvertX is a web file converter supporting hundreds of formats through tools like FFmpeg, ImageMagick, LibreOffice, Pandoc and Inkscape. This example passes `/dev/dri` through for hardware-accelerated video.

**Why you'd want it:** Convert files privately on your own hardware instead of uploading them to random websites.

| Item | Value |
|---|---|
| Stack folder | `/opt/stacks/convertx` |
| Category | Documents |
| Main image | `ghcr.io/c4illin/convertx:latest` |
| Services | 1 |
| Web UI | http://<SERVER_IP>:30013 |
| Official docs | https://github.com/C4illin/ConvertX |

## Requirements

- A Linux host with Docker Engine and the Docker Compose plugin (see [Docker](/docker))
- The shared folder layout and `.env`/`.secret` files from [Docker Folders](/dockerfolders) and [Environment](/environment)
- Optional: an Intel/AMD GPU at `/dev/dri` for VAAPI video conversion

## Compose

Folder layout:

```text
/opt/stacks/
├── _shared/
│   ├── .env        # shared, non-secret settings
│   └── .secret     # passwords, tokens and keys (chmod 600, never commit)
└── convertx/
    └── compose.yaml
/opt/appdata/convertx/   # persistent application data
```

`/opt/stacks/convertx/compose.yaml`:

```yaml
---
services:
  convertx:
    image: ghcr.io/c4illin/convertx:latest
    container_name: convertx
    hostname: convertx
    env_file:
      - /opt/stacks/_shared/.env
      - /opt/stacks/_shared/.secret
    environment:
      - ACCOUNT_REGISTRATION=false
      - HTTP_ALLOWED=true
      - ALLOW_UNAUTHENTICATED=true
      # JWT_SECRET comes from _shared/.secret via env_file
      - AUTO_DELETE_EVERY_N_HOURS=168
      - FFMPEG_ARGS=-hwaccel auto
      - LIBVA_DRIVER_NAME=iHD
      - HIDE_HISTORY=false
      - LANGUAGE=en
      - UNAUTHENTICATED_USER_SHARING=false
      - MAX_CONVERT_PROCESS=0
    logging:
      driver: json-file
      options:
        max-file: ${DOCKERLOGGING_MAXFILE:-10}  # Max number of log files
        max-size: ${DOCKERLOGGING_MAXSIZE:-200k}  # Max file size
    ports:
      - 30013:3000
    volumes:
      - /opt/appdata/convertx:/app/data:rw
    devices:
      - /dev/dri:/dev/dri
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 2000M
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

`/opt/stacks/_shared/.secret`:

```ini
JWT_SECRET=<RANDOM_STRING>
```

| Variable | File | Purpose | Example |
|---|---|---|---|
| `PUID` | `.env` | User ID the container runs as (see Docker Folders) | `1000` |
| `PGID` | `.env` | Group ID the container runs as | `1000` |
| `TZ` | `.env` | Time zone in tz database format, e.g. America/New_York | `<TZ>` |
| `DOCKERLOGGING_MAXFILE` | `.env` | Number of rotated container log files to keep | `10` |
| `DOCKERLOGGING_MAXSIZE` | `.env` | Maximum size of each container log file | `200k` |
| `JWT_SECRET` | `.secret` | Secret used to sign ConvertX sessions | `<RANDOM_STRING>` |

## Ports

| Host port | Container port | Protocol | Service | Purpose |
|---|---|---|---|---|
| 30013 | 3000 | tcp | `convertx` | Web UI |

## Volumes

| Host path | Container path | Service | Purpose |
|---|---|---|---|
| `/opt/appdata/convertx` | `/app/data` | `convertx` | Application data |

## Resource limits

Recommended limits, already set in the compose file under `deploy.resources`. Raise them if the app is starved, or lower them on small hosts.

| Service | CPU limit | Memory limit | Memory reservation | GPU |
|---|---|---|---|---|
| `convertx` | 2.0 | 2000M | - | - |

## Setup

1. Create the folders:
   ```bash
   sudo mkdir -p /opt/stacks/convertx /opt/appdata/convertx
   sudo chown -R $USER: /opt/stacks/convertx
   ```
2. Add the variables above to `/opt/stacks/_shared/.env` and `/opt/stacks/_shared/.secret` (`chmod 600 .secret`).
3. Save the compose file as `/opt/stacks/convertx/compose.yaml`.
4. Check the configuration, then start the stack:
   ```bash
   cd /opt/stacks/convertx
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret config --quiet
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
   ```
5. Watch the first start: `docker compose logs -f`.
6. Open `http://<SERVER_IP>:30013` and create the first account (registration is then closed by `ACCOUNT_REGISTRATION=false`).
7. Upload a file, pick a target format and convert.

> Tip: create a small wrapper such as `alias dc='docker compose --env-file /opt/stacks/_shared/.env --env-file /opt/stacks/_shared/.secret'` so every stack uses the same env files.
{.is-success}

## Updating

```bash
cd /opt/stacks/convertx
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret pull
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
docker image prune -f
```

Back up `/opt/appdata/convertx` before major upgrades. [Watchtower](/watchtower) can automate updates for stateless apps.

## Troubleshooting

- Check status and logs: `docker compose ps` and `docker compose logs --tail=100`.
- **Logged out on every restart:** set a fixed `JWT_SECRET` in `.secret`.
- **Video conversion is slow:** check `/dev/dri` permissions and `LIBVA_DRIVER_NAME` (`iHD` for newer Intel, `i965` for older, `radeonsi` for AMD).
- `ALLOW_UNAUTHENTICATED=true` is fine on a trusted LAN only. Set it to false if the app is reachable from outside.
- **A variable is empty or reported missing (`:?err`):** run the `config` command above to see which variable is not set in your env files.

## Related

- [Docker Stack Catalog](/docker-stacks)
- [Filestash](/filestash)
- [MeTube](/metube)

---
*Educational example. Review every path, port and credential before you deploy, and never publish your `.secret` file.*
