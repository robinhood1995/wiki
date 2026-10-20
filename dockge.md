---
title: Dockge
description: Dockge web UI for creating, editing and running Docker Compose stacks, with Docker Compose.
published: true
date: 2026-10-20T20:01:12.902Z
tags: docker, docker-compose, homelab, stack-management, compose, management, ui
editor: markdown
dateCreated: 2026-09-08T21:11:08.018Z
---

# Dockge

> Dockge web UI for creating, editing and running Docker Compose stacks, with Docker Compose.
{.is-info}

_Last updated: Oct 2026_

## Overview

Dockge is a lightweight, compose-first manager from the author of Uptime Kuma. It shows every stack in a folder, lets you edit `compose.yaml` in the browser, start, stop and update stacks, and watch logs live. Your files stay plain compose files on disk.

**Why you'd want it:** Manage all the stacks on this wiki from one page, without locking them into a proprietary format.

| Item | Value |
|---|---|
| Stack folder | `/opt/stacks/dockge` |
| Category | Stack management |
| Main image | `louislam/dockge:1` |
| Services | 1 |
| Web UI | http://<SERVER_IP>:5001 |
| Official docs | https://github.com/louislam/dockge |

## Requirements

- A Linux host with Docker Engine and the Docker Compose plugin (see [Docker](/docker))
- The shared folder layout and `.env`/`.secret` files from [Docker Folders](/dockerfolders) and [Environment](/environment)
- The stacks folder (`/opt/stacks`) created as in [Docker Folders](/dockerfolders)

## Compose

Folder layout:

```text
/opt/stacks/
├── _shared/
│   ├── .env        # shared, non-secret settings
│   └── .secret     # passwords, tokens and keys (chmod 600, never commit)
└── dockge/
    └── compose.yaml
/opt/appdata/dockge/   # persistent application data
```

`/opt/stacks/dockge/compose.yaml`:

```yaml
services:
  dockge:
    image: louislam/dockge:1
    container_name: dockge
    restart: unless-stopped
    hostname: dockge
    env_file:
      - /opt/stacks/_shared/.env
      - /opt/stacks/_shared/.secret
    environment:
      - DOCKGE_STACKS_DIR=/opt/stacks
    logging:
      driver: json-file
      options:
        max-file: ${DOCKERLOGGING_MAXFILE:-10}  # Max number of log files
        max-size: ${DOCKERLOGGING_MAXSIZE:-200k}  # Max file size
    ports:
      - 5001:5001
    volumes:
      # Docker socket access for orchestration
      - /var/run/docker.sock:/var/run/docker.sock
      # Dockge internal state database
      - /opt/appdata/dockge:/app/data
      # Stacks directory (MUST map 1:1 to host path)
      - /opt/stacks:/opt/stacks
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
| 5001 | 5001 | tcp | `dockge` | Web UI |

## Volumes

| Host path | Container path | Service | Purpose |
|---|---|---|---|
| `/var/run/docker.sock` | `/var/run/docker.sock` | `dockge` | Docker API access (grants control of Docker) |
| `/opt/appdata/dockge` | `/app/data` | `dockge` | Dockge accounts and settings |
| `/opt/stacks` | `/opt/stacks` | `dockge` | Your stacks folder (must be the same path inside and outside the container) |

## Resource limits

No limits are set. This app is light, so add `deploy.resources.limits` only if you need to cap it.

## Setup

1. Create the folders:
   ```bash
   sudo mkdir -p /opt/stacks/dockge /opt/appdata/dockge
   sudo chown -R $USER: /opt/stacks/dockge
   ```
2. Add the variables above to `/opt/stacks/_shared/.env` and `/opt/stacks/_shared/.secret` (`chmod 600 .secret`).
3. Save the compose file as `/opt/stacks/dockge/compose.yaml`.
4. Check the configuration, then start the stack:
   ```bash
   cd /opt/stacks/dockge
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret config --quiet
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
   ```
5. Watch the first start: `docker compose logs -f`.
6. Open `http://<SERVER_IP>:5001` and create the admin account.
7. Existing folders under `/opt/stacks` with a `compose.yaml` appear automatically.
8. Because these stacks use shared env files, add `env_file:` entries in each compose file (as shown on every stack page) rather than relying on `--env-file`.

> Tip: create a small wrapper such as `alias dc='docker compose --env-file /opt/stacks/_shared/.env --env-file /opt/stacks/_shared/.secret'` so every stack uses the same env files.
{.is-success}

## Updating

```bash
cd /opt/stacks/dockge
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret pull
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
docker image prune -f
```

Back up `/opt/appdata/dockge` before major upgrades. [Watchtower](/watchtower) can automate updates for stateless apps.

## Troubleshooting

- Check status and logs: `docker compose ps` and `docker compose logs --tail=100`.
- **Stacks fail with "no such file" for relative paths:** the stacks folder must be mounted at the same path inside the container as on the host (`/opt/stacks:/opt/stacks`) and `DOCKGE_STACKS_DIR` must match.
- **Stacks started with the CLI show as "not managed":** they are still fine; Dockge only manages folders inside `DOCKGE_STACKS_DIR`.
- The Docker socket gives full control of the host. Keep Dockge on your LAN or behind authentication.
- **A variable is empty or reported missing (`:?err`):** run the `config` command above to see which variable is not set in your env files.

## Related

- [Docker Stack Catalog](/docker-stacks)
- [Portainer](/portainer)
- [Watchtower](/watchtower)
- [Dozzle](/dozzle)

---
*Educational example. Review every path, port and credential before you deploy, and never publish your `.secret` file.*
