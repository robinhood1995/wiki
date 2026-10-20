---
title: Rackula
description: Rackula drag-and-drop rack layout planner with a persistent API backend, using Docker Compose.
published: true
date: 2026-10-20T20:01:42.057Z
tags: developer, docker, docker-compose, documentation, homelab, datacenter, planning, rack
editor: markdown
dateCreated: 2026-09-08T21:26:12.838Z
---

# Rackula

> Rackula drag-and-drop rack layout planner with a persistent API backend, using Docker Compose.
{.is-info}

_Last updated: Oct 2026_

## Overview

Rackula is a rack layout designer. Drag real devices from the NetBox device library into a rack, plan U positions, and export the layout to PNG, PDF or SVG or share it as a link. This stack runs the web app plus the optional API, which saves layouts on the server.

**Why you'd want it:** Plan where gear goes before you lift a 4U server, and keep an up-to-date picture of your rack.

| Item | Value |
|---|---|
| Stack folder | `/opt/stacks/rackula` |
| Category | Developer and IT tools |
| Main image | `ghcr.io/rackulalives/rackula:latest` |
| Services | 2 |
| Web UI | http://<SERVER_IP>:30010 |
| Official docs | https://github.com/RackulaLives/Rackula |

## Requirements

- A Linux host with Docker Engine and the Docker Compose plugin (see [Docker](/docker))
- The shared folder layout and `.env`/`.secret` files from [Docker Folders](/dockerfolders) and [Environment](/environment)

## Compose

Folder layout:

```text
/opt/stacks/
├── _shared/
│   ├── .env        # shared, non-secret settings
│   └── .secret     # passwords, tokens and keys (chmod 600, never commit)
└── rackula/
    └── compose.yaml
/opt/appdata/rackula/   # persistent application data
```

`/opt/stacks/rackula/compose.yaml`:

```yaml
services:
  rackula:
    image: ghcr.io/rackulalives/rackula:latest
    container_name: Rackula
    env_file:
      - /opt/stacks/_shared/.env
      - /opt/stacks/_shared/.secret
    healthcheck:
      test: ["CMD-SHELL", "nc -z 127.0.0.1 8080 || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 90s
    environment:
      - API_HOST=rackula-api
      - API_PORT=3001
      - RACKULA_LISTEN_PORT=8080
    logging:
      driver: json-file
      options:
        max-file: ${DOCKERLOGGING_MAXFILE:-10}  # Max number of log files
        max-size: ${DOCKERLOGGING_MAXSIZE:-200k}  # Max file size
    ports:
      - 30010:8080
    depends_on:
      - rackula-api
    restart: on-failure:5

  rackula-api:
    image: ghcr.io/rackulalives/rackula-api:latest
    container_name: Rackula-API
    user: "${PUID}:${PGID}"
    env_file:
      - /opt/stacks/_shared/.env
      - /opt/stacks/_shared/.secret
    volumes:
      - /opt/appdata/rackula:/data:rw
    environment:
      # CORS_ORIGIN comes from _shared/.env via env_file
      - DATA_DIR=/data
      - PORT=3001
    logging:
      driver: json-file
      options:
        max-file: ${DOCKERLOGGING_MAXFILE:-10}  # Max number of log files
        max-size: ${DOCKERLOGGING_MAXSIZE:-200k}  # Max file size
    restart: on-failure:5
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
| 30010 | 8080 | tcp | `rackula` | Web UI |

## Volumes

| Host path | Container path | Service | Purpose |
|---|---|---|---|
| `/opt/appdata/rackula` | `/data` | `rackula-api` | Application data |

## Resource limits

No limits are set. This app is light, so add `deploy.resources.limits` only if you need to cap it.

## Setup

1. Create the folders:
   ```bash
   sudo mkdir -p /opt/stacks/rackula /opt/appdata/rackula
   sudo chown -R $USER: /opt/stacks/rackula
   ```
2. Add the variables above to `/opt/stacks/_shared/.env` and `/opt/stacks/_shared/.secret` (`chmod 600 .secret`).
3. Save the compose file as `/opt/stacks/rackula/compose.yaml`.
4. Make the data folder writable by the API user: `sudo chown -R <PUID>:<PGID> /opt/appdata/rackula`.
5. Set `CORS_ORIGIN` in `.env` to the URL you open Rackula from.
6. Check the configuration, then start the stack:
   ```bash
   cd /opt/stacks/rackula
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret config --quiet
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
   ```
7. Watch the first start: `docker compose logs -f`.
8. Open `http://<SERVER_IP>:30010`, create a rack and drag devices into it.
9. For shared use, set `RACKULA_API_WRITE_TOKEN` (and optionally `RACKULA_AUTH_MODE=local` or `oidc`) as described in the upstream self-hosting guide.

> Tip: create a small wrapper such as `alias dc='docker compose --env-file /opt/stacks/_shared/.env --env-file /opt/stacks/_shared/.secret'` so every stack uses the same env files.
{.is-success}

## Updating

```bash
cd /opt/stacks/rackula
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret pull
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
docker image prune -f
```

Back up `/opt/appdata/rackula` before major upgrades. [Watchtower](/watchtower) can automate updates for stateless apps.

## Troubleshooting

- Check status and logs: `docker compose ps` and `docker compose logs --tail=100`.
- **Layouts not saved between browsers:** the API container is not reachable or `CORS_ORIGIN` does not match the page URL.
- **API exits with permission errors:** `/data` must be owned by the user in `user:` (the upstream default is 1001:1001).
- **Web container restarts:** `restart: on-failure:5` stops retrying after five failures; check `docker compose logs rackula-api` first.
- **A variable is empty or reported missing (`:?err`):** run the `config` command above to see which variable is not set in your env files.

## Related

- [Docker Stack Catalog](/docker-stacks)
- [diagrams.net (draw.io)](/drawio)
- [IT-Tools](/it-tools)

---
*Educational example. Review every path, port and credential before you deploy, and never publish your `.secret` file.*
