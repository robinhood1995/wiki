---
title: Plex
description: Plex Media Server (hotio image) with NVIDIA hardware transcoding and host networking, using Docker Compose.
published: true
date: 2026-10-18T03:28:21.169Z
tags: docker, docker-compose, gpu, homelab, media, media-server, transcoding
editor: markdown
dateCreated: 2026-09-08T21:11:08.018Z
---

# Plex

> Plex Media Server (hotio image) with NVIDIA hardware transcoding and host networking, using Docker Compose.
{.is-info}

_Last updated: Oct 2026_

## Overview

Plex organises your movies, TV, music and photos, and streams them to apps on TVs, phones and browsers. This example uses the hotio image, host networking, and an NVIDIA GPU for hardware transcoding (requires Plex Pass).

**Why you'd want it:** The most polished client apps and the easiest sharing with friends and family.

| Item | Value |
|---|---|
| Stack folder | `/opt/stacks/plex` |
| Category | Media |
| Main image | `ghcr.io/hotio/plex` |
| Services | 1 |
| Web UI | http://<SERVER_IP>:32400/web |
| Official docs | https://hotio.dev/containers/plex/ |

## Requirements

- A Linux host with Docker Engine and the Docker Compose plugin (see [Docker](/docker))
- The shared folder layout and `.env`/`.secret` files from [Docker Folders](/dockerfolders) and [Environment](/environment)
- A Plex account (Plex Pass for hardware transcoding)
- Media library at `/srv/media`
- Optional: NVIDIA GPU with the NVIDIA Container Toolkit, or Intel via `/dev/dri`

## Compose

Folder layout:

```text
/opt/stacks/
├── _shared/
│   ├── .env        # shared, non-secret settings
│   └── .secret     # passwords, tokens and keys (chmod 600, never commit)
└── plex/
    └── compose.yaml
/opt/appdata/plex/   # persistent application data
```

`/opt/stacks/plex/compose.yaml`:

```yaml
---
services:
  plex:
    image: ghcr.io/hotio/plex
    container_name: plex
    hostname: plex
    env_file:
      - /opt/stacks/_shared/.env
      - /opt/stacks/_shared/.secret
#    healthcheck:
#      test: curl --connect-timeout 15 --silent --show-error --fail http://127.0.0.1:32400/identity || exit 1
#      interval: 1m00s
#      timeout: 15s
#      retries: 3
#      start_period: 1m00s
    environment:
      - PLEX_SERVER=https://plex.<YOUR_DOMAIN>/
      # PLEX_CLAIM_TOKEN and PLEX_TOKEN come from _shared/.secret via env_file
      - PLEX_ADVERTISE_URL=${PLEX_ADVERTISE_URL}
      - PLEX_NO_AUTH_NETWORKS=
      - PLEX_BETA_INSTALL=${PLEX_BETA_INSTALL}
      - VERSION=docker # Specifies the Docker version of Plex
      - NVIDIA_VISIBLE_DEVICES=all     # Tells the NVIDIA Container Toolkit to expose all available NVIDIA GPUs to the container
      - NVIDIA_DRIVER_CAPABILITIES=all
      #- PLEX_DISABLE_HW_TRANSCODING=1 # for this comment out devices and tmpfs lines below. This will disable hardware transcoding and use software transcoding instead.
    logging:
      driver: json-file
      options:
        max-file: ${DOCKERLOGGING_MAXFILE:-10}  # Max number of log files
        max-size: ${DOCKERLOGGING_MAXSIZE:-200k}  # Max file size
    ports:
      - 32400:32400
      #- 8324:8324/tcp
      #- 32469:32469/tcp
      #- 1900:1900/udp
      #- 32410:32410/udp
      #- 32412:32412/udp
      #- 32413:32413/udp
      #- 32414:32414/udp
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /opt/appdata/plex:/config:rw
      - /srv/media:/data/media:rw
      #- /srv/photos:/media/photo:rw # Path to your photos, :ro for read-only :rw for read-write
    devices:                # optional: hardware transcoding (only with Plex Pass) uncomment this line.
      - /dev/dri:/dev/dri   # optional: hardware transcoding (only with Plex Pass) uncomment this line.
      #- /dev/dvb:/dev/dvb  # Uncomment this device for typical USB TV Tuner cards
    tmpfs:                  # optional: enable transcoding in RAM. uncomment this line.
      - /transcode          # optional: enable transcoding in RAM. uncomment this line.
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
    restart: unless-stopped
```

## Environment variables

Compose reads these from the shared files with `--env-file`. Replace every `<PLACEHOLDER>` with your own value.

`/opt/stacks/_shared/.env`:

```ini
PUID=1000
PGID=1000
TZ=<TZ>
PLEX_ADVERTISE_URL=http://<SERVER_IP>:32400
PLEX_BETA_INSTALL=false
DOCKERLOGGING_MAXFILE=10
DOCKERLOGGING_MAXSIZE=200k
```

`/opt/stacks/_shared/.secret`:

```ini
PLEX_CLAIM_TOKEN=<PLEX_CLAIM_TOKEN>
```

| Variable | File | Purpose | Example |
|---|---|---|---|
| `PUID` | `.env` | User ID the container runs as (see Docker Folders) | `1000` |
| `PGID` | `.env` | Group ID the container runs as | `1000` |
| `TZ` | `.env` | Time zone in tz database format, e.g. America/New_York | `<TZ>` |
| `PLEX_ADVERTISE_URL` | `.env` | URL(s) Plex advertises to clients | `http://<SERVER_IP>:32400` |
| `PLEX_BETA_INSTALL` | `.env` | Set to true to install Plex Pass beta builds | `false` |
| `DOCKERLOGGING_MAXFILE` | `.env` | Number of rotated container log files to keep | `10` |
| `DOCKERLOGGING_MAXSIZE` | `.env` | Maximum size of each container log file | `200k` |
| `PLEX_CLAIM_TOKEN` | `.secret` | One-time claim token from plex.tv/claim (valid 4 minutes) | `<PLEX_CLAIM_TOKEN>` |

## Ports

| Host port | Container port | Protocol | Service | Purpose |
|---|---|---|---|---|
| 32400 | 32400 | tcp | `plex` | Web UI |

## Volumes

| Host path | Container path | Service | Purpose |
|---|---|---|---|
| `/etc/localtime` | `/etc/localtime` | `plex` | Host time zone (read-only) |
| `/opt/appdata/plex` | `/config` | `plex` | Application configuration |
| `/srv/media` | `/data/media` | `plex` | Media library |

## Resource limits

Recommended limits, already set in the compose file under `deploy.resources`. Raise them if the app is starved, or lower them on small hosts.

| Service | CPU limit | Memory limit | Memory reservation | GPU |
|---|---|---|---|---|
| `plex` | 4.0 | 4000M | - | nvidia x all (gpu, utility, video) |

## Setup

1. Create the folders:
   ```bash
   sudo mkdir -p /opt/stacks/plex /opt/appdata/plex
   sudo chown -R $USER: /opt/stacks/plex
   ```
2. Add the variables above to `/opt/stacks/_shared/.env` and `/opt/stacks/_shared/.secret` (`chmod 600 .secret`).
3. Save the compose file as `/opt/stacks/plex/compose.yaml`.
4. Check the configuration, then start the stack:
   ```bash
   cd /opt/stacks/plex
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret config --quiet
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
   ```
5. Watch the first start: `docker compose logs -f`.
6. Get a claim token from https://plex.tv/claim and put it in `.secret` as `PLEX_CLAIM_TOKEN`. It expires after 4 minutes, so start the stack right away.
7. Open `http://<SERVER_IP>:32400/web` and finish setup.
8. Add libraries from `/data/media/...`.
9. Under **Settings > Transcoder**, enable hardware acceleration.

> Tip: create a small wrapper such as `alias dc='docker compose --env-file /opt/stacks/_shared/.env --env-file /opt/stacks/_shared/.secret'` so every stack uses the same env files.
{.is-success}

## Updating

```bash
cd /opt/stacks/plex
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret pull
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
docker image prune -f
```

Back up `/opt/appdata/plex` before major upgrades. [Watchtower](/watchtower) can automate updates for stateless apps.

## Troubleshooting

- Check status and logs: `docker compose ps` and `docker compose logs --tail=100`.
- **Server not claimed:** the token expired. Get a new one and recreate the container.
- **Remote access fails:** forward TCP 32400 on your router, or set `PLEX_ADVERTISE_URL` to the URL clients should use.
- **Use software transcoding:** set `PLEX_DISABLE_HW_TRANSCODING=1` and remove the `devices`, `runtime` and GPU lines.
- **A variable is empty or reported missing (`:?err`):** run the `config` command above to see which variable is not set in your env files.

## Related

- [Docker Stack Catalog](/docker-stacks)
- [Jellyfin](/jellyfin)
- [Seerr](/seerr)
- [Arr Stack](/arr-stack)
- [Tautulli](/tautulli)
- [Wizarr](/Wizarr)

---
*Educational example. Review every path, port and credential before you deploy, and never publish your `.secret` file.*
