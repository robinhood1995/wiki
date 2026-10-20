---
title: diagrams.net (draw.io)
description: Self-hosted diagrams.net (draw.io) diagram and network map editor with Docker Compose.
published: true
date: 2026-10-20T20:01:15.531Z
tags: docker, docker-compose, documents, homelab, diagrams, documentation, network-map
editor: markdown
dateCreated: 2026-09-08T21:15:35.866Z
---

# diagrams.net (draw.io)

> Self-hosted diagrams.net (draw.io) diagram and network map editor with Docker Compose.
{.is-info}

_Last updated: Oct 2026_

## Overview

diagrams.net (draw.io) is a full diagram editor for network maps, rack drawings, flowcharts and architecture diagrams. The self-hosted image runs entirely on your server; diagrams are saved to your own device or storage, not to the container.

**Why you'd want it:** Document your homelab network and services privately, with the same editor many companies use.

| Item | Value |
|---|---|
| Stack folder | `/opt/stacks/drawio` |
| Category | Documents |
| Main image | `jgraph/drawio` |
| Services | 1 |
| Web UI | http://<SERVER_IP>:30009 |
| Official docs | https://github.com/jgraph/docker-drawio |

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
└── drawio/
    └── compose.yaml
/opt/appdata/drawio/   # persistent application data
```

`/opt/stacks/drawio/compose.yaml`:

```yaml
---
services:
  drawio:
    image: jgraph/drawio
    container_name: drawio
    hostname: drawio
    env_file:
      - /opt/stacks/_shared/.env
      - /opt/stacks/_shared/.secret
    # PUBLIC_DNS comes from _shared/.env via env_file
    logging:
      driver: json-file
      options:
        max-file: ${DOCKERLOGGING_MAXFILE:-10}  # Max number of log files
        max-size: ${DOCKERLOGGING_MAXSIZE:-200k}  # Max file size
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:8080 || exit 1"]
      interval: 5s
      timeout: 5s
      retries: 5
      start_period: 10s
    ports:
      - 30009:8080
      #- 9443:8443
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
| 30009 | 8080 | tcp | `drawio` | Web UI |

## Volumes

No volumes.

## Resource limits

No limits are set. This app is light, so add `deploy.resources.limits` only if you need to cap it.

## Setup

1. Create the folders:
   ```bash
   sudo mkdir -p /opt/stacks/drawio /opt/appdata/drawio
   sudo chown -R $USER: /opt/stacks/drawio
   ```
2. Add the variables above to `/opt/stacks/_shared/.env` and `/opt/stacks/_shared/.secret` (`chmod 600 .secret`).
3. Save the compose file as `/opt/stacks/drawio/compose.yaml`.
4. Check the configuration, then start the stack:
   ```bash
   cd /opt/stacks/drawio
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret config --quiet
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
   ```
5. Watch the first start: `docker compose logs -f`.
6. Open `http://<SERVER_IP>:30009` and choose **Device** as the save location.
7. Use **File > Export as** for PNG, SVG or PDF to embed in this wiki or your docs.
8. Optional: put it behind your reverse proxy and set `PUBLIC_DNS` to that host name.

> Tip: create a small wrapper such as `alias dc='docker compose --env-file /opt/stacks/_shared/.env --env-file /opt/stacks/_shared/.secret'` so every stack uses the same env files.
{.is-success}

## Updating

```bash
cd /opt/stacks/drawio
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret pull
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
docker image prune -f
```

Back up `/opt/appdata/drawio` before major upgrades. [Watchtower](/watchtower) can automate updates for stateless apps.

## Troubleshooting

- Check status and logs: `docker compose ps` and `docker compose logs --tail=100`.
- **Healthcheck failing:** the check calls `curl` on port 8080 inside the container; wait for the start period or check `docker compose logs`.
- **Clipboard or file access blocked:** browsers allow these only on HTTPS or `localhost`; use a reverse proxy with TLS.
- The container keeps no diagrams. Save files to your device or a synced folder.
- **A variable is empty or reported missing (`:?err`):** run the `config` command above to see which variable is not set in your env files.

## Related

- [Docker Stack Catalog](/docker-stacks)
- [Rackula](/rackula)
- [File Browser](/filebrowser)

---
*Educational example. Review every path, port and credential before you deploy, and never publish your `.secret` file.*
