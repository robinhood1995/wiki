---
title: IT-Tools
description: IT-Tools collection of handy developer and sysadmin utilities in one web app, with Docker Compose.
published: true
date: 2026-10-20T20:01:28.826Z
tags: developer, docker, docker-compose, homelab, tools, utilities
editor: markdown
dateCreated: 2026-09-08T21:15:35.866Z
---

# IT-Tools

> IT-Tools collection of handy developer and sysadmin utilities in one web app, with Docker Compose.
{.is-info}

_Last updated: Oct 2026_

## Overview

IT-Tools bundles dozens of small utilities into one fast web page: UUID and password generators, hash and HMAC tools, Base64/URL encoders, JWT parser, cron expression helper, subnet calculator, QR code generator, JSON/YAML converters and more. Everything runs in your browser.

**Why you'd want it:** All the little tools you normally search the web for, private and always one bookmark away.

| Item | Value |
|---|---|
| Stack folder | `/opt/stacks/it-tools` |
| Category | Developer and IT tools |
| Main image | `corentinth/it-tools:latest` |
| Services | 1 |
| Web UI | http://<SERVER_IP>:30007 |
| Official docs | https://github.com/CorentinTh/it-tools |

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
└── it-tools/
    └── compose.yaml
/opt/appdata/it-tools/   # persistent application data
```

`/opt/stacks/it-tools/compose.yaml`:

```yaml
---
services:
  it-tools:
    image: corentinth/it-tools:latest
    container_name: it-tools
    hostname: itools
    env_file:
      - /opt/stacks/_shared/.env
      - /opt/stacks/_shared/.secret
    logging:
      driver: json-file
      options:
        max-file: ${DOCKERLOGGING_MAXFILE:-10}  # Max number of log files
        max-size: ${DOCKERLOGGING_MAXSIZE:-200k}  # Max file size
    ports:
      - 30007:80 # web UI on host port 30007
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
| 30007 | 80 | tcp | `it-tools` | web UI on host port 30007 |

## Volumes

No volumes.

## Resource limits

No limits are set. This app is light, so add `deploy.resources.limits` only if you need to cap it.

## Setup

1. Create the folders:
   ```bash
   sudo mkdir -p /opt/stacks/it-tools /opt/appdata/it-tools
   sudo chown -R $USER: /opt/stacks/it-tools
   ```
2. Add the variables above to `/opt/stacks/_shared/.env` and `/opt/stacks/_shared/.secret` (`chmod 600 .secret`).
3. Save the compose file as `/opt/stacks/it-tools/compose.yaml`.
4. Check the configuration, then start the stack:
   ```bash
   cd /opt/stacks/it-tools
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret config --quiet
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
   ```
5. Watch the first start: `docker compose logs -f`.
6. Open `http://<SERVER_IP>:30007`.
7. Star the tools you use most so they appear at the top.

> Tip: create a small wrapper such as `alias dc='docker compose --env-file /opt/stacks/_shared/.env --env-file /opt/stacks/_shared/.secret'` so every stack uses the same env files.
{.is-success}

## Updating

```bash
cd /opt/stacks/it-tools
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret pull
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
docker image prune -f
```

Back up `/opt/appdata/it-tools` before major upgrades. [Watchtower](/watchtower) can automate updates for stateless apps.

## Troubleshooting

- Check status and logs: `docker compose ps` and `docker compose logs --tail=100`.
- **Blank page after update:** clear the browser cache or do a hard refresh.
- **Some tools need HTTPS:** clipboard and camera features require a secure context; use a reverse proxy with TLS.
- **A variable is empty or reported missing (`:?err`):** run the `config` command above to see which variable is not set in your env files.

## Related

- [Docker Stack Catalog](/docker-stacks)
- [CyberChef](/cyberchef)
- [code-server](/code-server)
- [Adminer](/adminer)

---
*Educational example. Review every path, port and credential before you deploy, and never publish your `.secret` file.*
