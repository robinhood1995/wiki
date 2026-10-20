---
title: code-server
description: VS Code in the browser with code-server (LinuxServer) and Docker Compose.
published: true
date: 2026-10-18T03:27:41.874Z
tags: developer, docker, docker-compose, homelab, development, ide, vscode
editor: markdown
dateCreated: 2026-09-08T21:15:35.866Z
---

# code-server

> VS Code in the browser with code-server (LinuxServer) and Docker Compose.
{.is-info}

_Last updated: Oct 2026_

## Overview

code-server runs Visual Studio Code on your server and serves it to the browser. This example mounts `/opt/stacks` so you can edit all your compose files from one place.

**Why you'd want it:** Edit configs and scripts from any device with extensions, a terminal and Git, without SSH.

| Item | Value |
|---|---|
| Stack folder | `/opt/stacks/code-server` |
| Category | Developer and IT tools |
| Main image | `lscr.io/linuxserver/code-server:latest` |
| Services | 1 |
| Web UI | https://<SERVER_IP>:30012 |
| Official docs | https://docs.linuxserver.io/images/docker-code-server/ |

## Requirements

- A Linux host with Docker Engine and the Docker Compose plugin (see [Docker](/docker))
- The shared folder layout and `.env`/`.secret` files from [Docker Folders](/dockerfolders) and [Environment](/environment)
- Set `PASSWORD` or `HASHED_PASSWORD` in `.secret`, or put it behind an authenticating proxy

## Compose

Folder layout:

```text
/opt/stacks/
├── _shared/
│   ├── .env        # shared, non-secret settings
│   └── .secret     # passwords, tokens and keys (chmod 600, never commit)
└── code-server/
    └── compose.yaml
/opt/appdata/code-server/   # persistent application data
```

`/opt/stacks/code-server/compose.yaml`:

```yaml
---
services:
  code-server:
    image: lscr.io/linuxserver/code-server:latest
    container_name: code-server
    hostname: code-server
    env_file:
      - /opt/stacks/_shared/.env
      - /opt/stacks/_shared/.secret
    logging:
      driver: json-file
      options:
        max-file: ${DOCKERLOGGING_MAXFILE:-10}  # Max number of log files
        max-size: ${DOCKERLOGGING_MAXSIZE:-200k}  # Max file size
    volumes:
      - /opt/appdata/code-server/config:/config
      - /opt/appdata/code-server/:/code-server
      - /opt/stacks:/opt/stacks
    ports:
      - 30012:8443
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
PASSWORD=<STRONG_PASSWORD>
```

| Variable | File | Purpose | Example |
|---|---|---|---|
| `PUID` | `.env` | User ID the container runs as (see Docker Folders) | `1000` |
| `PGID` | `.env` | Group ID the container runs as | `1000` |
| `TZ` | `.env` | Time zone in tz database format, e.g. America/New_York | `<TZ>` |
| `DOCKERLOGGING_MAXFILE` | `.env` | Number of rotated container log files to keep | `10` |
| `DOCKERLOGGING_MAXSIZE` | `.env` | Maximum size of each container log file | `200k` |
| `PASSWORD` | `.secret` | Web UI password | `<STRONG_PASSWORD>` |

## Ports

| Host port | Container port | Protocol | Service | Purpose |
|---|---|---|---|---|
| 30012 | 8443 | tcp | `code-server` | Web UI |

## Volumes

| Host path | Container path | Service | Purpose |
|---|---|---|---|
| `/opt/appdata/code-server/config` | `/config` | `code-server` | Application configuration |
| `/opt/appdata/code-server/` | `/code-server` | `code-server` | Persistent data |
| `/opt/stacks` | `/opt/stacks` | `code-server` | Persistent data |

## Resource limits

Recommended limits, already set in the compose file under `deploy.resources`. Raise them if the app is starved, or lower them on small hosts.

| Service | CPU limit | Memory limit | Memory reservation | GPU |
|---|---|---|---|---|
| `code-server` | 2.0 | 2000M | - | - |

## Setup

1. Create the folders:
   ```bash
   sudo mkdir -p /opt/stacks/code-server /opt/appdata/code-server
   sudo chown -R $USER: /opt/stacks/code-server
   ```
2. Add the variables above to `/opt/stacks/_shared/.env` and `/opt/stacks/_shared/.secret` (`chmod 600 .secret`).
3. Save the compose file as `/opt/stacks/code-server/compose.yaml`.
4. Check the configuration, then start the stack:
   ```bash
   cd /opt/stacks/code-server
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret config --quiet
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
   ```
5. Watch the first start: `docker compose logs -f`.
6. Open `https://<SERVER_IP>:30012` and accept the self-signed certificate.
7. Sign in with the password from `.secret`.
8. Open `/opt/stacks` as a folder and install the Docker and YAML extensions.

> Tip: create a small wrapper such as `alias dc='docker compose --env-file /opt/stacks/_shared/.env --env-file /opt/stacks/_shared/.secret'` so every stack uses the same env files.
{.is-success}

## Updating

```bash
cd /opt/stacks/code-server
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret pull
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
docker image prune -f
```

Back up `/opt/appdata/code-server` before major upgrades. [Watchtower](/watchtower) can automate updates for stateless apps.

## Troubleshooting

- Check status and logs: `docker compose ps` and `docker compose logs --tail=100`.
- **Permission denied editing files:** `PUID`/`PGID` must own the mounted folders.
- **Extensions missing after update:** keep `/config` on persistent storage.
- **A variable is empty or reported missing (`:?err`):** run the `config` command above to see which variable is not set in your env files.

## Related

- [Docker Stack Catalog](/docker-stacks)
- [Dockge](/dockge)
- [CyberChef](/cyberchef)
- [IT-Tools](/it-tools)

---
*Educational example. Review every path, port and credential before you deploy, and never publish your `.secret` file.*
