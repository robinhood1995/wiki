---
title: Double Commander
description: Double Commander dual-pane file manager in the browser (LinuxServer KasmVNC) with Docker Compose.
published: true
date: 2026-10-18T03:27:54.966Z
tags: docker, docker-compose, documents, file-manager, files, homelab, kasmvnc
editor: markdown
dateCreated: 2026-09-08T21:26:12.838Z
---

# Double Commander

> Double Commander dual-pane file manager in the browser (LinuxServer KasmVNC) with Docker Compose.
{.is-info}

_Last updated: Oct 2026_

## Overview

Double Commander is a two-pane file manager, similar to Total Commander, running as a desktop app in your browser through KasmVNC. It is handy for moving large folders on the server itself without dragging data over the network.

**Why you'd want it:** Fast server-side copies and moves with a familiar dual-pane interface.

| Item | Value |
|---|---|
| Stack folder | `/opt/stacks/doublecommander` |
| Category | Documents |
| Main image | `lscr.io/linuxserver/doublecommander:latest` |
| Services | 1 |
| Web UI | http://<SERVER_IP>:30021 |
| Official docs | https://docs.linuxserver.io/images/docker-doublecommander/ |

## Requirements

- A Linux host with Docker Engine and the Docker Compose plugin (see [Docker](/docker))
- The shared folder layout and `.env`/`.secret` files from [Docker Folders](/dockerfolders) and [Environment](/environment)
- `DOUBLECOMMANDER_PASSWORD` set in `.secret` (the stack refuses to start without it)

## Compose

Folder layout:

```text
/opt/stacks/
├── _shared/
│   ├── .env        # shared, non-secret settings
│   └── .secret     # passwords, tokens and keys (chmod 600, never commit)
└── doublecommander/
    └── compose.yaml
/opt/appdata/doublecommander/   # persistent application data
```

`/opt/stacks/doublecommander/compose.yaml`:

```yaml
services:
  doublecommander:
    image: lscr.io/linuxserver/doublecommander:latest
    container_name: double-commander
    cap_drop:
      - NET_RAW
      - NET_ADMIN
    env_file:
      - /opt/stacks/_shared/.env
      - /opt/stacks/_shared/.secret
    environment:
      - CUSTOM_USER=admin
      - PASSWORD=${DOUBLECOMMANDER_PASSWORD:?err}
    logging:
      driver: json-file
      options:
        max-file: ${DOCKERLOGGING_MAXFILE:-10}  # Max number of log files
        max-size: ${DOCKERLOGGING_MAXSIZE:-200k}  # Max file size
    ports:
      - 30021:3000
    security_opt:
      - no-new-privileges:true
    volumes:
      - /opt/appdata/doublecommander/config:/config:rw
      - /opt/appdata/doublecommander/data:/data:rw
      - /srv:/srv:rw
    shm_size: 1gb # Temp Memory
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 2000M
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [compute,video,graphics,utility]
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

`/opt/stacks/_shared/.secret`:

```ini
DOUBLECOMMANDER_PASSWORD=<STRONG_PASSWORD>
```

| Variable | File | Purpose | Example |
|---|---|---|---|
| `PUID` | `.env` | User ID the container runs as (see Docker Folders) | `1000` |
| `PGID` | `.env` | Group ID the container runs as | `1000` |
| `TZ` | `.env` | Time zone in tz database format, e.g. America/New_York | `<TZ>` |
| `DOUBLECOMMANDER_PASSWORD` | `.secret` | Password for the Double Commander web GUI | `<STRONG_PASSWORD>` |
| `DOCKERLOGGING_MAXFILE` | `.env` | Number of rotated container log files to keep | `10` |
| `DOCKERLOGGING_MAXSIZE` | `.env` | Maximum size of each container log file | `200k` |

## Ports

| Host port | Container port | Protocol | Service | Purpose |
|---|---|---|---|---|
| 30021 | 3000 | tcp | `doublecommander` | Web UI |

## Volumes

| Host path | Container path | Service | Purpose |
|---|---|---|---|
| `/opt/appdata/doublecommander/config` | `/config` | `doublecommander` | Application configuration |
| `/opt/appdata/doublecommander/data` | `/data` | `doublecommander` | Application data |
| `/srv` | `/srv` | `doublecommander` | Persistent data |

## Resource limits

Recommended limits, already set in the compose file under `deploy.resources`. Raise them if the app is starved, or lower them on small hosts.

| Service | CPU limit | Memory limit | Memory reservation | GPU |
|---|---|---|---|---|
| `doublecommander` | 2.0 | 2000M | - | nvidia x 1 (compute, video, graphics, utility) |

## Setup

1. Create the folders:
   ```bash
   sudo mkdir -p /opt/stacks/doublecommander /opt/appdata/doublecommander
   sudo chown -R $USER: /opt/stacks/doublecommander
   ```
2. Add the variables above to `/opt/stacks/_shared/.env` and `/opt/stacks/_shared/.secret` (`chmod 600 .secret`).
3. Save the compose file as `/opt/stacks/doublecommander/compose.yaml`.
4. Check the configuration, then start the stack:
   ```bash
   cd /opt/stacks/doublecommander
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret config --quiet
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
   ```
5. Watch the first start: `docker compose logs -f`.
6. Open `http://<SERVER_IP>:30021` and sign in as `admin` with your password.
7. Browse `/srv` in one pane and your target in the other.

> Tip: create a small wrapper such as `alias dc='docker compose --env-file /opt/stacks/_shared/.env --env-file /opt/stacks/_shared/.secret'` so every stack uses the same env files.
{.is-success}

## Updating

```bash
cd /opt/stacks/doublecommander
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret pull
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
docker image prune -f
```

Back up `/opt/appdata/doublecommander` before major upgrades. [Watchtower](/watchtower) can automate updates for stateless apps.

## Troubleshooting

- Check status and logs: `docker compose ps` and `docker compose logs --tail=100`.
- **`DOUBLECOMMANDER_PASSWORD` error on start:** the `:?err` guard means the variable is missing from `.secret`.
- The `/srv` mount is read-write; mount it `:ro` if you only need to browse.
- No NVIDIA GPU? Remove the `reservations.devices` block.
- **A variable is empty or reported missing (`:?err`):** run the `config` command above to see which variable is not set in your env files.

## Related

- [Docker Stack Catalog](/docker-stacks)
- [Filestash](/filestash)
- [File Browser](/filebrowser)

---
*Educational example. Review every path, port and credential before you deploy, and never publish your `.secret` file.*
