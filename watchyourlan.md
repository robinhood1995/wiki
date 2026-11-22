---
title: WatchYourLAN
description: ARP-based LAN device discovery and inventory with WatchYourLAN, using Docker Compose.
published: true
date: 2026-11-22T06:33:52.037Z
tags: docker, docker-compose, homelab, monitoring, network
editor: markdown
dateCreated: 2026-11-22T06:33:52.037Z
---

# WatchYourLAN

> Lightweight ARP-based LAN device discovery and inventory (WatchYourLAN), using Docker Compose.
{.is-info}

_Last updated: Nov 2026_

## Overview

WatchYourLAN watches a network interface, notices devices as they appear, and keeps a simple inventory with names, MACs and last-seen times. It is useful for spotting unexpected equipment on a home or lab network.

**Why you'd want it:** A quiet, always-on list of what is on your LAN, without a heavy NMS.

| Item | Value |
|---|---|
| Stack folder | `/opt/stacks/watchyourlan` |
| Category | Monitoring |
| Main image | `ghcr.io/aceberg/watchyourlan:latest` |
| Services | 1 |
| Web UI | http://<SERVER_IP>:8840 |
| Official docs | https://github.com/aceberg/WatchYourLAN |

## Requirements

- A Linux host with Docker Engine and the Docker Compose plugin (see [Docker](/docker))
- The shared folder layout and `.env`/`.secret` files from [Docker Folders](/dockerfolders) and [Environment](/environment)
- A network interface that can see the LAN you want to scan (often requires host networking)
- The interface name on the host, which you will put in place of `<INTERFACE>`

## Compose

Folder layout:

```text
/opt/stacks/
├── _shared/
│   ├── .env        # shared, non-secret settings
│   └── .secret     # passwords, tokens and keys (chmod 600, never commit)
└── watchyourlan/
    └── compose.yaml
/opt/appdata/watchyourlan/   # persistent application data
```

`/opt/stacks/watchyourlan/compose.yaml`:

```yaml
---
services:
  watchyourlan:
    image: ghcr.io/aceberg/watchyourlan:latest
    container_name: watchyourlan
    hostname: watchyourlan
    env_file:
      - /opt/stacks/_shared/.env
      - /opt/stacks/_shared/.secret
    environment:
      - TZ=${TZ}
      - IFACE=<INTERFACE>
      - GUIIP=0.0.0.0
      - GUIPORT=8840
      - TIMEOUT=60
    logging:
      driver: json-file
      options:
        max-file: ${DOCKERLOGGING_MAXFILE:-10}
        max-size: ${DOCKERLOGGING_MAXSIZE:-200k}
    network_mode: host
    volumes:
      - /opt/appdata/watchyourlan:/data/WatchYourLAN
    restart: unless-stopped
```

Host networking is used so ARP discovery can see your LAN. The UI then listens on port 8840 on the host. Do not also publish `8840:8840` when `network_mode: host` is set.

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
| PUID | .env | User ID the container runs as (see Docker Folders) | 1000 |
| PGID | .env | Group ID the container runs as | 1000 |
| TZ | .env | Time zone in tz database format, e.g. America/New_York | <TZ> |
| DOCKERLOGGING_MAXFILE | .env | Number of rotated container log files to keep | 10 |
| DOCKERLOGGING_MAXSIZE | .env | Maximum size of each container log file | 200k |
| IFACE | compose | Host interface to watch | <INTERFACE> |

Replace `<INTERFACE>` with the interface on your host (the name shown by `ip -br link`). If you have several VLANs, check the upstream docs for listing more than one interface.

## Ports

| Host port | Container port | Protocol | Service | Purpose |
|---|---|---|---|---|
| 8840 | 8840 | tcp | watchyourlan | Web UI (host network) |

## Volumes

| Host path | Container path | Service | Purpose |
|---|---|---|---|
| `/opt/appdata/watchyourlan` | `/data/WatchYourLAN` | watchyourlan | Inventory database and config |

## Resource limits

No limits are set. This app is light, so add `deploy.resources.limits` only if you need to cap it.

## Setup

1. Create the folders:
   ```bash
   sudo mkdir -p /opt/stacks/watchyourlan /opt/appdata/watchyourlan
   sudo chown -R $USER: /opt/stacks/watchyourlan
   ```
2. Add the variables above to `/opt/stacks/_shared/.env` and `/opt/stacks/_shared/.secret` (`chmod 600 .secret`).
3. Save the compose file as `/opt/stacks/watchyourlan/compose.yaml`.
4. Check the configuration, then start the stack:
   ```bash
   cd /opt/stacks/watchyourlan
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret config --quiet
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
   ```
5. Watch the first start: `docker compose logs -f`.

6. Set `<INTERFACE>` to the LAN interface you want to watch, then recreate the container if you had to edit the file after the first start.
7. Open `http://<SERVER_IP>:8840` and confirm devices you expect appear after a minute or two.
8. Keep this UI off the public internet; it shows your device inventory. Put it behind [oauth2-proxy](/oauth2-proxy) if you need remote access.

> Tip: create a small wrapper such as `alias dc='docker compose --env-file /opt/stacks/_shared/.env --env-file /opt/stacks/_shared/.secret'` so every stack uses the same env files.
{.is-success}
## Updating

```bash
cd /opt/stacks/watchyourlan
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret pull
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
docker image prune -f
```

Back up `/opt/appdata/watchyourlan` before major upgrades. [Watchtower](/watchtower) can automate updates for stateless apps.

## Troubleshooting

- Check status and logs: `docker compose ps` and `docker compose logs --tail=100`.
- **Empty inventory:** confirm `<INTERFACE>` is the correct name and that the host, not a bridge-only interface, can see LAN traffic. Host networking is required for ARP discovery in this example.
- **Port 8840 already in use:** another service is bound to that port on the host. Change `GUIPORT` and open the new port instead.
- **A variable is empty or reported missing (`:?err`):** run the `config` command above to see which variable is not set in your env files.

## Related

- [Docker Stack Catalog](/docker-stacks)
- [Prometheus](/prometheus)
- [Netdata](/netdata)
- [oauth2-proxy](/oauth2-proxy)

---
*Educational example. Review every path, port and credential before you deploy, and never publish your `.secret` file.*
