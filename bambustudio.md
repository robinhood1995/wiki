---
title: Bambu Studio
description: Bambu Studio 3D-printer slicer running in the browser (LinuxServer KasmVNC) with Docker Compose.
published: true
date: 2026-10-18T03:27:39.247Z
tags: docker, docker-compose, homelab, 3d-printing, desktop, developer, kasmvnc
editor: markdown
dateCreated: 2026-09-08T21:26:12.838Z
---

# Bambu Studio

> Bambu Studio 3D-printer slicer running in the browser (LinuxServer KasmVNC) with Docker Compose.
{.is-info}

_Last updated: Oct 2026_

## Overview

Bambu Studio is the slicer for Bambu Lab 3D printers. The LinuxServer image runs the full desktop app in a browser tab via KasmVNC, so you can slice from any computer, tablet or Chromebook.

**Why you'd want it:** One always-on slicer with your profiles, reachable from every device.

| Item | Value |
|---|---|
| Stack folder | `/opt/stacks/bambustudio` |
| Category | Developer and IT tools |
| Main image | `lscr.io/linuxserver/bambustudio:latest` |
| Services | 1 |
| Web UI | http://<SERVER_IP>:30018 |
| Official docs | https://docs.linuxserver.io/images/docker-bambustudio/ |

## Requirements

- A Linux host with Docker Engine and the Docker Compose plugin (see [Docker](/docker))
- The shared folder layout and `.env`/`.secret` files from [Docker Folders](/dockerfolders) and [Environment](/environment)
- 4 CPU cores and 4 GB RAM recommended for slicing large models

## Compose

Folder layout:

```text
/opt/stacks/
├── _shared/
│   ├── .env        # shared, non-secret settings
│   └── .secret     # passwords, tokens and keys (chmod 600, never commit)
└── bambustudio/
    └── compose.yaml
/opt/appdata/bambustudio/   # persistent application data
```

`/opt/stacks/bambustudio/compose.yaml`:

```yaml
---
services:
  bambustudio:
    image: lscr.io/linuxserver/bambustudio:latest
    container_name: bambustudio
    hostname: bambustudio
    env_file:
      - /opt/stacks/_shared/.env
      - /opt/stacks/_shared/.secret
    environment:
      # Optional: set to true for dark mode
      - DARK_MODE=true
    logging:
      driver: json-file
      options:
        max-file: ${DOCKERLOGGING_MAXFILE:-10}  # Max number of log files
        max-size: ${DOCKERLOGGING_MAXSIZE:-200k}  # Max file size
    volumes:
      # Map the config directory to a local path
      - /opt/appdata/bambustudio/config:/config
    ports:
      # Map host ports to container ports (GUI access)
      - 30018:3000
      #- 3001:3001
    shm_size: "1gb" # Recommended to increase shared memory size
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: '4.0'
          memory: 4000M
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
| 30018 | 3000 | tcp | `bambustudio` | Web UI |

## Volumes

| Host path | Container path | Service | Purpose |
|---|---|---|---|
| `/opt/appdata/bambustudio/config` | `/config` | `bambustudio` | Application configuration |

## Resource limits

Recommended limits, already set in the compose file under `deploy.resources`. Raise them if the app is starved, or lower them on small hosts.

| Service | CPU limit | Memory limit | Memory reservation | GPU |
|---|---|---|---|---|
| `bambustudio` | 4.0 | 4000M | - | - |

## Setup

1. Create the folders:
   ```bash
   sudo mkdir -p /opt/stacks/bambustudio /opt/appdata/bambustudio
   sudo chown -R $USER: /opt/stacks/bambustudio
   ```
2. Add the variables above to `/opt/stacks/_shared/.env` and `/opt/stacks/_shared/.secret` (`chmod 600 .secret`).
3. Save the compose file as `/opt/stacks/bambustudio/compose.yaml`.
4. Check the configuration, then start the stack:
   ```bash
   cd /opt/stacks/bambustudio
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret config --quiet
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
   ```
5. Watch the first start: `docker compose logs -f`.
6. Open `http://<SERVER_IP>:30018`.
7. Sign in to your Bambu account or add your printer in LAN mode.
8. Keep project files under `/config` so they persist.

> Tip: create a small wrapper such as `alias dc='docker compose --env-file /opt/stacks/_shared/.env --env-file /opt/stacks/_shared/.secret'` so every stack uses the same env files.
{.is-success}

## Updating

```bash
cd /opt/stacks/bambustudio
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret pull
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
docker image prune -f
```

Back up `/opt/appdata/bambustudio` before major upgrades. [Watchtower](/watchtower) can automate updates for stateless apps.

## Troubleshooting

- Check status and logs: `docker compose ps` and `docker compose logs --tail=100`.
- **Black screen or crashes:** raise `shm_size` (1 GB is the minimum recommended).
- **Slow UI:** use a wired connection, or lower the stream quality in the KasmVNC side panel.
- **Do not expose it to the internet without authentication.** It is a full desktop session.
- **A variable is empty or reported missing (`:?err`):** run the `config` command above to see which variable is not set in your env files.

## Related

- [Docker Stack Catalog](/docker-stacks)
- [Double Commander](/doublecommander)
- [code-server](/code-server)

---
*Educational example. Review every path, port and credential before you deploy, and never publish your `.secret` file.*
