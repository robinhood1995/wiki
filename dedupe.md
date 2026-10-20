---
title: JellEmPlex Dedupe
description: JellEmPlex Dedupe web tool for finding and removing duplicate movies in Jellyfin, Emby and Plex, with Docker Compose.
published: true
date: 2026-10-20T20:01:10.217Z
tags: docker, docker-compose, homelab, jellyfin, media, plex, duplicates, emby
editor: markdown
dateCreated: 2026-09-08T21:26:12.838Z
---

# JellEmPlex Dedupe

> JellEmPlex Dedupe web tool for finding and removing duplicate movies in Jellyfin, Emby and Plex, with Docker Compose.
{.is-info}

_Last updated: Oct 2026_

## Overview

JellEmPlex Dedupe connects to your Jellyfin, Emby or Plex server and finds movies that exist more than once (same title and year). It shows resolution, codecs and file size side by side, so you can keep the best copy and delete the rest from the web UI.

**Why you'd want it:** Reclaim disk space and tidy libraries after years of upgrades and re-downloads.

| Item | Value |
|---|---|
| Stack folder | `/opt/stacks/dedupe` |
| Category | Media |
| Main image | `spaceinvaderone/jellemplex-dedupe:latest` |
| Services | 1 |
| Web UI | http://<SERVER_IP>:3369 |
| Official docs | https://github.com/SpaceinvaderOne/jellemplex-dedupe |

## Requirements

- A Linux host with Docker Engine and the Docker Compose plugin (see [Docker](/docker))
- The shared folder layout and `.env`/`.secret` files from [Docker Folders](/dockerfolders) and [Environment](/environment)
- A Jellyfin or Emby API key (Dashboard > API Keys), or a Plex token (`PlexOnlineToken` in `Preferences.xml`)

## Compose

Folder layout:

```text
/opt/stacks/
├── _shared/
│   ├── .env        # shared, non-secret settings
│   └── .secret     # passwords, tokens and keys (chmod 600, never commit)
└── dedupe/
    └── compose.yaml
/opt/appdata/dedupe/   # persistent application data
```

`/opt/stacks/dedupe/compose.yaml`:

```yaml
---
services:
  jellyemplex-dedupe:
    image: spaceinvaderone/jellemplex-dedupe:latest
    container_name: dedupe
    hostname: dedupe
    env_file:
      - /opt/stacks/_shared/.env
      - /opt/stacks/_shared/.secret
    environment:
      - DELETE_FILES=true   # allow deleting duplicates from the web UI
    volumes:
      - /opt/appdata/dedupe/config:/config
      - /srv/media:/media   # optional: your media library
    logging:
      driver: json-file
      options:
        max-file: ${DOCKERLOGGING_MAXFILE:-10}  # Max number of log files
        max-size: ${DOCKERLOGGING_MAXSIZE:-200k}  # Max file size
    ports:
      - 3369:80
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
| 3369 | 80 | tcp | `jellyemplex-dedupe` | Web UI |

## Volumes

| Host path | Container path | Service | Purpose |
|---|---|---|---|
| `/opt/appdata/dedupe/config` | `/config` | `jellyemplex-dedupe` | Saved server configurations |
| `/srv/media` | `/media` | `jellyemplex-dedupe` | optional: your media library |

## Resource limits

No limits are set. This app is light, so add `deploy.resources.limits` only if you need to cap it.

## Setup

1. Create the folders:
   ```bash
   sudo mkdir -p /opt/stacks/dedupe /opt/appdata/dedupe
   sudo chown -R $USER: /opt/stacks/dedupe
   ```
2. Add the variables above to `/opt/stacks/_shared/.env` and `/opt/stacks/_shared/.secret` (`chmod 600 .secret`).
3. Save the compose file as `/opt/stacks/dedupe/compose.yaml`.
4. Check the configuration, then start the stack:
   ```bash
   cd /opt/stacks/dedupe
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret config --quiet
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
   ```
5. Watch the first start: `docker compose logs -f`.
6. Open `http://<SERVER_IP>:3369`.
7. Choose the server type, enter `http://<SERVER_IP>:8096` (Jellyfin) or `http://<SERVER_IP>:32400` (Plex) and paste the API key or token.
8. Click **Find Duplicate Movies**, review the results, then delete the copies you do not want.

> Tip: create a small wrapper such as `alias dc='docker compose --env-file /opt/stacks/_shared/.env --env-file /opt/stacks/_shared/.secret'` so every stack uses the same env files.
{.is-success}

## Updating

```bash
cd /opt/stacks/dedupe
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret pull
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
docker image prune -f
```

Back up `/opt/appdata/dedupe` before major upgrades. [Watchtower](/watchtower) can automate updates for stateless apps.

## Troubleshooting

- Check status and logs: `docker compose ps` and `docker compose logs --tail=100`.
- **Deletes are permanent:** review carefully, and keep backups of anything irreplaceable.
- **No results:** the tool matches movies by name and year; make sure your library metadata is correct.
- **Connection refused:** use the server IP, not `localhost`, since the tool runs in its own container.
- **A variable is empty or reported missing (`:?err`):** run the `config` command above to see which variable is not set in your env files.

## Related

- [Docker Stack Catalog](/docker-stacks)
- [Jellyfin](/jellyfin)
- [Plex](/plex)
- [Tautulli](/tautulli)

---
*Educational example. Review every path, port and credential before you deploy, and never publish your `.secret` file.*
