---
title: CyberChef
description: CyberChef, the "cyber Swiss army knife", self-hosted with Docker Compose.
published: true
date: 2026-10-18T03:27:47.074Z
tags: developer, docker, docker-compose, homelab, encoding, security, tools
editor: markdown
dateCreated: 2026-09-08T21:15:35.866Z
---

# CyberChef

> CyberChef, the "cyber Swiss army knife", self-hosted with Docker Compose.
{.is-info}

_Last updated: Oct 2026_

## Overview

CyberChef is a web app for encoding, decoding, hashing, compressing, parsing and transforming data, with hundreds of operations you chain into "recipes". Everything runs client-side in the browser.

**Why you'd want it:** Decode Base64, JWTs, hex dumps and timestamps without pasting sensitive data into public sites.

| Item | Value |
|---|---|
| Stack folder | `/opt/stacks/cyberchef` |
| Category | Developer and IT tools |
| Main image | `ghcr.io/tamcore/cyberchef:latest` |
| Services | 1 |
| Web UI | http://<SERVER_IP>:30003 |
| Official docs | https://github.com/gchq/CyberChef |

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
└── cyberchef/
    └── compose.yaml
/opt/appdata/cyberchef/   # persistent application data
```

`/opt/stacks/cyberchef/compose.yaml`:

```yaml
---
services:
  cyberchef:
    image: ghcr.io/tamcore/cyberchef:latest # Or another suitable image like gchq/cyberchef
    container_name: cyberchef
    hostname: cyberchef
    env_file:
      - /opt/stacks/_shared/.env
      - /opt/stacks/_shared/.secret
    environment:
      - DEFAULT_THEME=auto # Optional: auto, light, or dark
    logging:
      driver: json-file
      options:
        max-file: ${DOCKERLOGGING_MAXFILE:-10}  # Max number of log files
        max-size: ${DOCKERLOGGING_MAXSIZE:-200k}  # Max file size
    ports:
      - 30003:8080/tcp # Maps host port 8080 to container port 8080
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
| 30003 | 8080 | tcp | `cyberchef` | Maps host port 8080 to container port 8080 |

## Volumes

No volumes.

## Resource limits

No limits are set. This app is light, so add `deploy.resources.limits` only if you need to cap it.

## Setup

1. Create the folders:
   ```bash
   sudo mkdir -p /opt/stacks/cyberchef /opt/appdata/cyberchef
   sudo chown -R $USER: /opt/stacks/cyberchef
   ```
2. Add the variables above to `/opt/stacks/_shared/.env` and `/opt/stacks/_shared/.secret` (`chmod 600 .secret`).
3. Save the compose file as `/opt/stacks/cyberchef/compose.yaml`.
4. Check the configuration, then start the stack:
   ```bash
   cd /opt/stacks/cyberchef
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret config --quiet
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
   ```
5. Watch the first start: `docker compose logs -f`.
6. Open `http://<SERVER_IP>:30003`.
7. Drag operations into the recipe and bookmark useful recipes. The URL stores them.

> Tip: create a small wrapper such as `alias dc='docker compose --env-file /opt/stacks/_shared/.env --env-file /opt/stacks/_shared/.secret'` so every stack uses the same env files.
{.is-success}

## Updating

```bash
cd /opt/stacks/cyberchef
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret pull
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
docker image prune -f
```

Back up `/opt/appdata/cyberchef` before major upgrades. [Watchtower](/watchtower) can automate updates for stateless apps.

## Troubleshooting

- Check status and logs: `docker compose ps` and `docker compose logs --tail=100`.
- **Blank page:** clear the browser cache after an image update.
- **A variable is empty or reported missing (`:?err`):** run the `config` command above to see which variable is not set in your env files.

## Related

- [Docker Stack Catalog](/docker-stacks)
- [IT-Tools](/it-tools)
- [code-server](/code-server)

---
*Educational example. Review every path, port and credential before you deploy, and never publish your `.secret` file.*
