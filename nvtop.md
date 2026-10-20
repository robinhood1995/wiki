---
title: nvtop
description: nvtop interactive GPU process monitor (htop for GPUs) in a container, with Docker Compose.
published: true
date: 2026-10-20T20:01:36.755Z
tags: docker, docker-compose, gpu, homelab, monitoring, nvidia
editor: markdown
dateCreated: 2026-09-08T21:26:12.838Z
---

# nvtop

> nvtop interactive GPU process monitor (htop for GPUs) in a container, with Docker Compose.
{.is-info}

_Last updated: Oct 2026_

## Overview

nvtop is a terminal tool like `htop` for GPUs. It shows utilisation, memory, temperature, power and the processes using each GPU, with live graphs. Running it in a container means you do not need to install it on the host.

**Why you'd want it:** See at a glance which container (Plex, Jellyfin, Frigate, AI) is using your GPU right now.

| Item | Value |
|---|---|
| Stack folder | `/opt/stacks/nvtop` |
| Category | Monitoring |
| Main image | `69guitar1015/nvtop:latest` |
| Services | 1 |
| Web UI | Terminal: `docker attach nvtop` (detach with Ctrl+P, Ctrl+Q) |
| Official docs | https://github.com/Syllo/nvtop |

## Requirements

- A Linux host with Docker Engine and the Docker Compose plugin (see [Docker](/docker))
- The shared folder layout and `.env`/`.secret` files from [Docker Folders](/dockerfolders) and [Environment](/environment)
- An NVIDIA GPU with the driver installed on the host
- The NVIDIA Container Toolkit

## Compose

Folder layout:

```text
/opt/stacks/
├── _shared/
│   ├── .env        # shared, non-secret settings
│   └── .secret     # passwords, tokens and keys (chmod 600, never commit)
└── nvtop/
    └── compose.yaml
/opt/appdata/nvtop/   # persistent application data
```

`/opt/stacks/nvtop/compose.yaml`:

```yaml
---
services:
  nvtop:
    image: 69guitar1015/nvtop:latest   # community pre-built image
    container_name: nvtop
    hostname: nvtop
    env_file:
      - /opt/stacks/_shared/.env
      - /opt/stacks/_shared/.secret
    # Alternative: build from source if you prefer
#    build:
#       context: https://github.com/Syllo/nvtop.git
#       dockerfile: Dockerfile
    # Required for NVIDIA GPU access in Docker Compose
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all          # or specify '1' / device_ids: ['0']
              capabilities: [gpu, utility, compute]

    # Alternative older syntax (if the above doesn't work on your version):
    # runtime: nvidia
    # environment:
    #   - NVIDIA_VISIBLE_DEVICES=all
    #   - NVIDIA_DRIVER_CAPABILITIES=all

    # Interactive TUI + host PID namespace for best GPU monitoring
    stdin_open: true
    tty: true
    pid: host

    # Optional: privileged if you hit permission issues (rarely needed)
    # privileged: true
    restart: unless-stopped
```

## Environment variables

Compose reads these from the shared files with `--env-file`. Replace every `<PLACEHOLDER>` with your own value.

`/opt/stacks/_shared/.env`:

```ini
PUID=1000
PGID=1000
TZ=<TZ>
```

| Variable | File | Purpose | Example |
|---|---|---|---|
| `PUID` | `.env` | User ID the container runs as (see Docker Folders) | `1000` |
| `PGID` | `.env` | Group ID the container runs as | `1000` |
| `TZ` | `.env` | Time zone in tz database format, e.g. America/New_York | `<TZ>` |

## Ports

This stack publishes no ports.

## Volumes

No volumes.

## Resource limits

Recommended limits, already set in the compose file under `deploy.resources`. Raise them if the app is starved, or lower them on small hosts.

| Service | CPU limit | Memory limit | Memory reservation | GPU |
|---|---|---|---|---|
| `nvtop` | - | - | - | nvidia x all (gpu, utility, compute) |

## Setup

1. Create the folders:
   ```bash
   sudo mkdir -p /opt/stacks/nvtop /opt/appdata/nvtop
   sudo chown -R $USER: /opt/stacks/nvtop
   ```
2. Add the variables above to `/opt/stacks/_shared/.env` and `/opt/stacks/_shared/.secret` (`chmod 600 .secret`).
3. Save the compose file as `/opt/stacks/nvtop/compose.yaml`.
4. Check the configuration, then start the stack:
   ```bash
   cd /opt/stacks/nvtop
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret config --quiet
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
   ```
5. Watch the first start: `docker compose logs -f`.
6. Start the stack, then run `docker attach nvtop` to see the live view.
7. Detach without stopping it: **Ctrl+P**, then **Ctrl+Q**.
8. Or run it on demand instead: `docker compose run --rm nvtop`.

> Tip: create a small wrapper such as `alias dc='docker compose --env-file /opt/stacks/_shared/.env --env-file /opt/stacks/_shared/.secret'` so every stack uses the same env files.
{.is-success}

## Updating

```bash
cd /opt/stacks/nvtop
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret pull
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
docker image prune -f
```

Back up `/opt/appdata/nvtop` before major upgrades. [Watchtower](/watchtower) can automate updates for stateless apps.

## Troubleshooting

- Check status and logs: `docker compose ps` and `docker compose logs --tail=100`.
- **"No GPU to monitor":** check `nvidia-smi` on the host and that the NVIDIA runtime is configured for Docker.
- **No process names:** `pid: host` is required to see processes from other containers.
- **Typing `q` stopped it:** that exits nvtop and the container restarts; use the detach keys instead.
- **A variable is empty or reported missing (`:?err`):** run the `config` command above to see which variable is not set in your env files.

## Related

- [Docker Stack Catalog](/docker-stacks)
- [DCGM Exporter](/dcgm-exporter)
- [Netdata](/netdata)
- [Grafana](/grafana)

---
*Educational example. Review every path, port and credential before you deploy, and never publish your `.secret` file.*
