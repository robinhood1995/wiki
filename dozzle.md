---
title: Dozzle
description: Real-time Docker container log viewer with Dozzle and Docker Compose.
published: true
date: 2026-10-18T03:27:57.523Z
tags: docker, docker-compose, homelab, monitoring, logs
editor: markdown
dateCreated: 2026-09-08T21:15:35.866Z
---

# Dozzle

> Real-time Docker container log viewer with Dozzle and Docker Compose.
{.is-info}

_Last updated: Oct 2026_

## Overview

Dozzle streams container logs to your browser in real time, with search, split views and multi-host support through lightweight agents.

**Why you'd want it:** Check logs from your phone without SSH or `docker logs`.

| Item | Value |
|---|---|
| Stack folder | `/opt/stacks/dozzle` |
| Category | Monitoring |
| Main image | `amir20/dozzle:latest` |
| Services | 1 |
| Web UI | http://<SERVER_IP>:38000 |
| Official docs | https://dozzle.dev/ |

## Requirements

- A Linux host with Docker Engine and the Docker Compose plugin (see [Docker](/docker))
- The shared folder layout and `.env`/`.secret` files from [Docker Folders](/dockerfolders) and [Environment](/environment)
- Optional: a Dozzle agent (`amir20/dozzle agent`) on each other Docker host, on port 7007

## Compose

Folder layout:

```text
/opt/stacks/
├── _shared/
│   ├── .env        # shared, non-secret settings
│   └── .secret     # passwords, tokens and keys (chmod 600, never commit)
└── dozzle/
    └── compose.yaml
/opt/appdata/dozzle/   # persistent application data
```

`/opt/stacks/dozzle/compose.yaml`:

```yaml
---
services:
  dozzle:
    image: amir20/dozzle:latest
    container_name: dozzle
    hostname: dozzle
    env_file:
      - /opt/stacks/_shared/.env
      - /opt/stacks/_shared/.secret
    environment:
      - DOZZLE_REMOTE_AGENT=<AGENT_HOST_IP>:7007
      - DOZZLE_HOSTNAME=<SERVER_NAME>
    logging:
      driver: json-file
      options:
        max-file: ${DOCKERLOGGING_MAXFILE:-10}  # Max number of log files
        max-size: ${DOCKERLOGGING_MAXSIZE:-200k}  # Max file size
    ports:
     - 38000:8080
    volumes:
     - /var/run/docker.sock:/var/run/docker.sock
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
| 38000 | 8080 | tcp | `dozzle` | Web UI |

## Volumes

| Host path | Container path | Service | Purpose |
|---|---|---|---|
| `/var/run/docker.sock` | `/var/run/docker.sock` | `dozzle` | Docker API access (grants control of Docker) |

## Resource limits

No limits are set. This app is light, so add `deploy.resources.limits` only if you need to cap it.

## Setup

1. Create the folders:
   ```bash
   sudo mkdir -p /opt/stacks/dozzle /opt/appdata/dozzle
   sudo chown -R $USER: /opt/stacks/dozzle
   ```
2. Add the variables above to `/opt/stacks/_shared/.env` and `/opt/stacks/_shared/.secret` (`chmod 600 .secret`).
3. Save the compose file as `/opt/stacks/dozzle/compose.yaml`.
4. Check the configuration, then start the stack:
   ```bash
   cd /opt/stacks/dozzle
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret config --quiet
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
   ```
5. Watch the first start: `docker compose logs -f`.
6. Open `http://<SERVER_IP>:38000`.
7. Set `DOZZLE_HOSTNAME` to the name you want shown for this host.
8. For other hosts, run the agent there and list it in `DOZZLE_REMOTE_AGENT` (comma-separated `host:7007`).

> Tip: create a small wrapper such as `alias dc='docker compose --env-file /opt/stacks/_shared/.env --env-file /opt/stacks/_shared/.secret'` so every stack uses the same env files.
{.is-success}

## Updating

```bash
cd /opt/stacks/dozzle
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret pull
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
docker image prune -f
```

Back up `/opt/appdata/dozzle` before major upgrades. [Watchtower](/watchtower) can automate updates for stateless apps.

## Troubleshooting

- Check status and logs: `docker compose ps` and `docker compose logs --tail=100`.
- **Remote host missing:** port 7007 must be reachable from the Dozzle host.
- The Docker socket gives full control of Docker; protect Dozzle with authentication if it is reachable beyond your LAN.
- **A variable is empty or reported missing (`:?err`):** run the `config` command above to see which variable is not set in your env files.

## Related

- [Docker Stack Catalog](/docker-stacks)
- [Netdata](/netdata)
- [Dockge](/dockge)
- [Watchtower](/watchtower)

---
*Educational example. Review every path, port and credential before you deploy, and never publish your `.secret` file.*
