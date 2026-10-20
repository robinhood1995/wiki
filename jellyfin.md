---
title: Jellyfin
description: Jellyfin media server with NVIDIA hardware transcoding and host networking, using Docker Compose.
published: true
date: 2026-10-18T03:28:08.095Z
tags: docker, docker-compose, gpu, homelab, media, media-server, transcoding
editor: markdown
dateCreated: 2026-09-08T21:11:08.018Z
---

# Jellyfin

> Jellyfin media server with NVIDIA hardware transcoding and host networking, using Docker Compose.
{.is-info}

_Last updated: Oct 2026_

## Overview

Jellyfin is a free, open-source media server for movies, TV, music and live TV. It has no accounts or paywalls, and hardware transcoding is included. This example uses host networking for DLNA and client discovery, and an NVIDIA GPU for transcoding.

**Why you'd want it:** A fully free Plex alternative that you control end to end.

| Item | Value |
|---|---|
| Stack folder | `/opt/stacks/jellyfin` |
| Category | Media |
| Main image | `jellyfin/jellyfin:latest` |
| Services | 1 |
| Web UI | http://<SERVER_IP>:8096 |
| Official docs | https://jellyfin.org/docs/ |

## Requirements

- A Linux host with Docker Engine and the Docker Compose plugin (see [Docker](/docker))
- The shared folder layout and `.env`/`.secret` files from [Docker Folders](/dockerfolders) and [Environment](/environment)
- Media library mounted at `/srv/media`
- Optional: NVIDIA GPU with the NVIDIA Container Toolkit, or Intel/AMD via `/dev/dri`

## Compose

Folder layout:

```text
/opt/stacks/
├── _shared/
│   ├── .env        # shared, non-secret settings
│   └── .secret     # passwords, tokens and keys (chmod 600, never commit)
└── jellyfin/
    └── compose.yaml
/opt/appdata/jellyfin/   # persistent application data
```

`/opt/stacks/jellyfin/compose.yaml`:

```yaml
---
services:
  jellyfin:
    image: jellyfin/jellyfin:latest
    container_name: jellyfin
    hostname: jellyfin
    env_file:
      - /opt/stacks/_shared/.env
      - /opt/stacks/_shared/.secret
    environment:
      - NVIDIA_VISIBLE_DEVICES=all     # Tells the NVIDIA Container Toolkit to expose all available NVIDIA GPUs to the container
      - NVIDIA_DRIVER_CAPABILITIES=all
      - ROC_ENABLE_PRE_VEGA=1 # Hardware Acceleration
    logging:
      driver: json-file
      options:
        max-file: ${DOCKERLOGGING_MAXFILE:-10}  # Max number of log files
        max-size: ${DOCKERLOGGING_MAXSIZE:-200k}  # Max file size
    healthcheck:
      test: timeout 10s bash -c ':> /dev/tcp/127.0.0.1/8096' || exit 1
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 90s
    volumes:
      - /opt/appdata/jellyfin/config:/config # Path to your config directory
      - /opt/appdata/jellyfin/cache:/cache   # Path to your cache directory
      - /srv/media:/data/media:rw
      #- /srv/photos/library:/photos:rw # Path to your photos, :ro for read-only :rw for read-write
      - /dev/shm:/data/transcode # optional: enable transcoding in RAM. uncomment this line.
    devices:                # optional: hardware transcoding uncomment this line.
      - /dev/dri:/dev/dri   # optional: hardware transcoding uncomment this line.
    #ports:
      #- 8096:8096 # Jellyfin Web UI
      #- 8920:8920 # Jellyfin DNLA (optional)
      #- 7359:7359/udp # Jellyfin DLNA discovery
      #- 1900:1900/udp # Jellyfin DLNA advertising
    runtime: nvidia # Specify NVIDIA runtime for GPU access
    deploy:
      resources:
        limits:
          cpus: '4.0'
          memory: 4000M
        reservations:
          devices:
            - driver: nvidia
              count: all # Use all available NVIDIA GPUs
              capabilities: [gpu, utility, video] # Specify required capabilities
    network_mode: host
    #sysctls:
      #- net.ipv6.conf.all.disable_ipv6=1
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
| host network | - | - | `jellyfin` | 8096/tcp (web UI), 8920/tcp (HTTPS), 7359/udp + 1900/udp (discovery/DLNA) |

## Volumes

| Host path | Container path | Service | Purpose |
|---|---|---|---|
| `/opt/appdata/jellyfin/config` | `/config` | `jellyfin` | Path to your config directory |
| `/opt/appdata/jellyfin/cache` | `/cache` | `jellyfin` | Path to your cache directory |
| `/srv/media` | `/data/media` | `jellyfin` | Media library |
| `/dev/shm` | `/data/transcode` | `jellyfin` | optional: enable transcoding in RAM. uncomment this line. |

## Resource limits

Recommended limits, already set in the compose file under `deploy.resources`. Raise them if the app is starved, or lower them on small hosts.

| Service | CPU limit | Memory limit | Memory reservation | GPU |
|---|---|---|---|---|
| `jellyfin` | 4.0 | 4000M | - | nvidia x all (gpu, utility, video) |

## Setup

1. Create the folders:
   ```bash
   sudo mkdir -p /opt/stacks/jellyfin /opt/appdata/jellyfin
   sudo chown -R $USER: /opt/stacks/jellyfin
   ```
2. Add the variables above to `/opt/stacks/_shared/.env` and `/opt/stacks/_shared/.secret` (`chmod 600 .secret`).
3. Save the compose file as `/opt/stacks/jellyfin/compose.yaml`.
4. Check the configuration, then start the stack:
   ```bash
   cd /opt/stacks/jellyfin
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret config --quiet
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
   ```
5. Watch the first start: `docker compose logs -f`.
6. Open `http://<SERVER_IP>:8096` and run the setup wizard.
7. Add libraries pointing to `/data/media/movies`, `/data/media/tv` and so on.
8. Under **Dashboard > Playback > Transcoding**, choose NVENC (NVIDIA) or VAAPI/QSV (Intel) and enable the codecs your GPU supports.

> Tip: create a small wrapper such as `alias dc='docker compose --env-file /opt/stacks/_shared/.env --env-file /opt/stacks/_shared/.secret'` so every stack uses the same env files.
{.is-success}

## Updating

```bash
cd /opt/stacks/jellyfin
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret pull
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
docker image prune -f
```

Back up `/opt/appdata/jellyfin` before major upgrades. [Watchtower](/watchtower) can automate updates for stateless apps.

## Troubleshooting

- Check status and logs: `docker compose ps` and `docker compose logs --tail=100`.
- **Transcoding falls back to CPU:** run `nvidia-smi` inside the container (`docker exec jellyfin nvidia-smi`).
- **Clients can't discover the server:** host networking is required for DLNA and auto-discovery.
- No NVIDIA GPU? Remove `runtime: nvidia` and the `reservations.devices` block.
- **A variable is empty or reported missing (`:?err`):** run the `config` command above to see which variable is not set in your env files.

## Related

- [Docker Stack Catalog](/docker-stacks)
- [Plex](/plex)
- [Seerr](/seerr)
- [Arr Stack](/arr-stack)
- [Wizarr](/Wizarr)

---
*Educational example. Review every path, port and credential before you deploy, and never publish your `.secret` file.*
