---
title: Watchtower
description: Watchtower automatic nightly container image updates with exclusions and notifications, using Docker Compose.
published: true
date: 2026-10-18T03:28:31.484Z
tags: docker, docker-compose, homelab, maintenance, stack-management, updates
editor: markdown
dateCreated: 2026-09-08T21:15:35.866Z
---

# Watchtower

> Watchtower automatic nightly container image updates with exclusions and notifications, using Docker Compose.
{.is-info}

_Last updated: Oct 2026_

## Overview

Watchtower watches your running containers and, on a schedule, pulls newer images and recreates the containers with the same settings. This example runs nightly at 02:00, cleans up old images, sends notifications, and excludes containers you should upgrade by hand.

**Why you'd want it:** Hands-off security and feature updates for the bulk of your stacks.

| Item | Value |
|---|---|
| Stack folder | `/opt/stacks/watchtower` |
| Category | Stack management |
| Main image | `containrrr/watchtower:latest` |
| Services | 1 |
| Web UI | None (HTTP API on 8080 inside the container, optional) |
| Official docs | https://containrrr.dev/watchtower/ |

## Requirements

- A Linux host with Docker Engine and the Docker Compose plugin (see [Docker](/docker))
- The shared folder layout and `.env`/`.secret` files from [Docker Folders](/dockerfolders) and [Environment](/environment)
- A notification target for update reports (email, Discord, Gotify, etc. via Shoutrrr)

## Compose

Folder layout:

```text
/opt/stacks/
├── _shared/
│   ├── .env        # shared, non-secret settings
│   └── .secret     # passwords, tokens and keys (chmod 600, never commit)
└── watchtower/
    └── compose.yaml
/opt/appdata/watchtower/   # persistent application data
```

`/opt/stacks/watchtower/compose.yaml`:

```yaml
services:
  watchtower:
    image: containrrr/watchtower:latest
    container_name: watchtower
    hostname: watchtower
    env_file:
      - /opt/stacks/_shared/.env
      - /opt/stacks/_shared/.secret
    environment:
      - WATCHTOWER_DISABLE_CONTAINERS=immich_postgres,immich_redis,frigate # pinned or stateful containers you update by hand
      - WATCHTOWER_SCHEDULE=0 2 * * *
      - WATCHTOWER_CLEANUP=true
      #- WATCHTOWER_NOTIFICATION_URL= comes from _shared/.secret via env_file
      - WATCHTOWER_NOTIFICATION_EMAIL_DELAY=2
      - WATCHTOWER_NOTIFICATIONS_HOSTNAME=<SERVER_NAME>
      - WATCHTOWER_NO_STARTUP_MESSAGE=true
      # WATCHTOWER_HTTP_API_TOKEN comes from _shared/.secret via env_file
      - WATCHTOWER_HTTP_API_UPDATE=true
      - WATCHTOWER_HTTP_API_PERIODIC_POLLS=true
      - WATCHTOWER_TIMEOUT=30s
      - DOCKER_API_VERSION=1.44
    logging:
      driver: json-file
      options:
        max-file: ${DOCKERLOGGING_MAXFILE:-10}  # Max number of log files
        max-size: ${DOCKERLOGGING_MAXSIZE:-200k}  # Max file size
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    restart: always
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
WATCHTOWER_HTTP_API_TOKEN=<RANDOM_TOKEN>
WATCHTOWER_NOTIFICATION_URL=<SHOUTRRR_URL>
```

| Variable | File | Purpose | Example |
|---|---|---|---|
| `PUID` | `.env` | User ID the container runs as (see Docker Folders) | `1000` |
| `PGID` | `.env` | Group ID the container runs as | `1000` |
| `TZ` | `.env` | Time zone in tz database format, e.g. America/New_York | `<TZ>` |
| `DOCKERLOGGING_MAXFILE` | `.env` | Number of rotated container log files to keep | `10` |
| `DOCKERLOGGING_MAXSIZE` | `.env` | Maximum size of each container log file | `200k` |
| `WATCHTOWER_HTTP_API_TOKEN` | `.secret` | Token for the Watchtower HTTP API | `<RANDOM_TOKEN>` |
| `WATCHTOWER_NOTIFICATION_URL` | `.secret` | Shoutrrr notification URL (email, Discord, etc.) | `<SHOUTRRR_URL>` |

## Ports

This stack publishes no ports.

## Volumes

| Host path | Container path | Service | Purpose |
|---|---|---|---|
| `/var/run/docker.sock` | `/var/run/docker.sock` | `watchtower` | Docker API access (grants control of Docker) |

## Resource limits

No limits are set. This app is light, so add `deploy.resources.limits` only if you need to cap it.

## Setup

1. Create the folders:
   ```bash
   sudo mkdir -p /opt/stacks/watchtower /opt/appdata/watchtower
   sudo chown -R $USER: /opt/stacks/watchtower
   ```
2. Add the variables above to `/opt/stacks/_shared/.env` and `/opt/stacks/_shared/.secret` (`chmod 600 .secret`).
3. Save the compose file as `/opt/stacks/watchtower/compose.yaml`.
4. Check the configuration, then start the stack:
   ```bash
   cd /opt/stacks/watchtower
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret config --quiet
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
   ```
5. Watch the first start: `docker compose logs -f`.
6. List containers that must not auto-update in `WATCHTOWER_DISABLE_CONTAINERS`: databases, Immich Postgres/Redis, and pinned apps like Frigate.
7. Set `WATCHTOWER_NOTIFICATION_URL` in `.secret`.
8. Start the stack. The schedule `0 2 * * *` uses the container time zone (`TZ`).

> Tip: create a small wrapper such as `alias dc='docker compose --env-file /opt/stacks/_shared/.env --env-file /opt/stacks/_shared/.secret'` so every stack uses the same env files.
{.is-success}

## Updating

```bash
cd /opt/stacks/watchtower
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret pull
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
docker image prune -f
```

Back up `/opt/appdata/watchtower` before major upgrades. [Watchtower](/watchtower) can automate updates for stateless apps.

## Troubleshooting

- Check status and logs: `docker compose ps` and `docker compose logs --tail=100`.
- **"client version is too old":** set `DOCKER_API_VERSION` to match your Docker Engine (`docker version`).
- **An app broke after an update:** pin its image tag, add it to the exclusion list, and roll back with the previous tag.
- **A variable is empty or reported missing (`:?err`):** run the `config` command above to see which variable is not set in your env files.

## Related

- [Docker Stack Catalog](/docker-stacks)
- [Frigate NVR](/frigate)
- [Immich](/immich)
- [Dockge](/dockge)

---
*Educational example. Review every path, port and credential before you deploy, and never publish your `.secret` file.*
