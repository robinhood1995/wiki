---
title: MeTube
description: MeTube web UI for yt-dlp video and audio downloads with Docker Compose.
published: true
date: 2026-10-18T03:28:15.917Z
tags: docker, docker-compose, downloads, homelab, media, youtube, yt-dlp
editor: markdown
dateCreated: 2026-09-08T21:15:35.866Z
---

# MeTube

> MeTube web UI for yt-dlp video and audio downloads with Docker Compose.
{.is-info}

_Last updated: Oct 2026_

## Overview

MeTube is a web front end for yt-dlp. Paste a link from YouTube or any of the thousand-plus supported sites, pick a quality or audio-only, and it downloads to your server.

**Why you'd want it:** Save videos for offline viewing straight into your media library.

| Item | Value |
|---|---|
| Stack folder | `/opt/stacks/metube` |
| Category | Media |
| Main image | `ghcr.io/alexta69/metube` |
| Services | 1 |
| Web UI | http://<SERVER_IP>:38001 |
| Official docs | https://github.com/alexta69/metube |

## Requirements

- A Linux host with Docker Engine and the Docker Compose plugin (see [Docker](/docker))
- The shared folder layout and `.env`/`.secret` files from [Docker Folders](/dockerfolders) and [Environment](/environment)
- A download folder, e.g. `/srv/media/metube`

## Compose

Folder layout:

```text
/opt/stacks/
├── _shared/
│   ├── .env        # shared, non-secret settings
│   └── .secret     # passwords, tokens and keys (chmod 600, never commit)
└── metube/
    └── compose.yaml
/opt/appdata/metube/   # persistent application data
```

`/opt/stacks/metube/compose.yaml`:

```yaml
---
services:
  metube:
    image: ghcr.io/alexta69/metube
    container_name: metube
    env_file:
      - /opt/stacks/_shared/.env
      - /opt/stacks/_shared/.secret
    environment:
      - DEFAULT_THEME=auto # Optional: auto, light, or dark
    logging:
      driver: json-file
      options:
        max-file: ${DOCKERLOGGING_MAXFILE:-10}  # Max number of log files
        max-size: ${DOCKERLOGGING_MAXSIZE:-200k}  # Max file size
    ports:
      - "38001:8081" # Host_Port:Container_Port
    volumes:
      - /srv/media/metube:/downloads # Host_Path:Container_Path
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
| 38001 | 8081 | tcp | `metube` | Host_Port:Container_Port |

## Volumes

| Host path | Container path | Service | Purpose |
|---|---|---|---|
| `/srv/media/metube` | `/downloads` | `metube` | Host_Path:Container_Path |

## Resource limits

Recommended limits, already set in the compose file under `deploy.resources`. Raise them if the app is starved, or lower them on small hosts.

| Service | CPU limit | Memory limit | Memory reservation | GPU |
|---|---|---|---|---|
| `metube` | 2.0 | 2000M | - | - |

## Setup

1. Create the folders:
   ```bash
   sudo mkdir -p /opt/stacks/metube /opt/appdata/metube
   sudo chown -R $USER: /opt/stacks/metube
   ```
2. Add the variables above to `/opt/stacks/_shared/.env` and `/opt/stacks/_shared/.secret` (`chmod 600 .secret`).
3. Save the compose file as `/opt/stacks/metube/compose.yaml`.
4. Check the configuration, then start the stack:
   ```bash
   cd /opt/stacks/metube
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret config --quiet
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
   ```
5. Watch the first start: `docker compose logs -f`.
6. Open `http://<SERVER_IP>:38001`, paste a URL and click **Add**.
7. Optional: add the folder to Jellyfin or Plex as a library.
8. Install the MeTube browser extension for one-click sending.

> Tip: create a small wrapper such as `alias dc='docker compose --env-file /opt/stacks/_shared/.env --env-file /opt/stacks/_shared/.secret'` so every stack uses the same env files.
{.is-success}

## Updating

```bash
cd /opt/stacks/metube
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret pull
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
docker image prune -f
```

Back up `/opt/appdata/metube` before major upgrades. [Watchtower](/watchtower) can automate updates for stateless apps.

## Troubleshooting

- Check status and logs: `docker compose ps` and `docker compose logs --tail=100`.
- **Downloads fail for one site:** update the image. yt-dlp changes often.
- **Files owned by root:** set `UID`/`GID` (or `PUID`/`PGID`) to your user.
- **A variable is empty or reported missing (`:?err`):** run the `config` command above to see which variable is not set in your env files.

## Related

- [Docker Stack Catalog](/docker-stacks)
- [Jellyfin](/jellyfin)
- [Plex](/plex)
- [ConvertX](/convertx)

---
*Educational example. Review every path, port and credential before you deploy, and never publish your `.secret` file.*
