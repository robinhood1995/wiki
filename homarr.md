---
title: Homarr
description: Customizable homelab dashboard with app shortcuts and widgets, using Docker Compose.
published: true
date: 2026-11-22T06:33:56.414Z
tags: docker, docker-compose, homelab, dashboard
editor: markdown
dateCreated: 2026-11-22T06:33:56.414Z
---

# Homarr

> Customizable homelab dashboard (Homarr) with app shortcuts, status and widgets, using Docker Compose.
{.is-info}

_Last updated: Nov 2026_

## Overview

Homarr is a browser start page for your lab. You pin apps, see whether they are up, and add widgets such as calendars or media-request status. It is a convenient home page, not a replacement for each app's own UI.

**Why you'd want it:** One bookmark that reaches every service you run.

| Item | Value |
|---|---|
| Stack folder | `/opt/stacks/homarr` |
| Category | Stack management |
| Main image | `ghcr.io/homarr-labs/homarr:latest` |
| Services | 1 |
| Web UI | http://<SERVER_IP>:7575 |
| Official docs | https://homarr.dev/docs/getting-started/installation/docker |

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
└── homarr/
    └── compose.yaml
/opt/appdata/homarr/   # persistent application data
```

`/opt/stacks/homarr/compose.yaml`:

```yaml
---
services:
  homarr:
    image: ghcr.io/homarr-labs/homarr:latest
    container_name: homarr
    hostname: homarr
    env_file:
      - /opt/stacks/_shared/.env
      - /opt/stacks/_shared/.secret
    environment:
      - TZ=${TZ}
      - SECRET_ENCRYPTION_KEY=${SECRET_ENCRYPTION_KEY:?err}
    logging:
      driver: json-file
      options:
        max-file: ${DOCKERLOGGING_MAXFILE:-10}
        max-size: ${DOCKERLOGGING_MAXSIZE:-200k}
    ports:
      - 7575:7575
    volumes:
      - /opt/appdata/homarr/appdata:/appdata
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

`/opt/stacks/_shared/.secret` (chmod 600):

```ini
SECRET_ENCRYPTION_KEY=<ENCRYPTION_KEY>
```

Generate the key with `openssl rand -hex 32` and paste the 64-character hex string. If you lose it, Homarr cannot decrypt stored secrets.

| Variable | File | Purpose | Example |
|---|---|---|---|
| PUID | .env | User ID the container runs as (see Docker Folders) | 1000 |
| PGID | .env | Group ID the container runs as | 1000 |
| TZ | .env | Time zone in tz database format, e.g. America/New_York | <TZ> |
| DOCKERLOGGING_MAXFILE | .env | Number of rotated container log files to keep | 10 |
| DOCKERLOGGING_MAXSIZE | .env | Maximum size of each container log file | 200k |
| SECRET_ENCRYPTION_KEY | .secret | 64-character hex key for stored secrets | <ENCRYPTION_KEY> |

## Ports

| Host port | Container port | Protocol | Service | Purpose |
|---|---|---|---|---|
| 7575 | 7575 | tcp | homarr | Web UI |

## Volumes

| Host path | Container path | Service | Purpose |
|---|---|---|---|
| `/opt/appdata/homarr/appdata` | `/appdata` | homarr | Dashboards, apps and encrypted secrets |

## Resource limits

No limits are set. This app is light, so add `deploy.resources.limits` only if you need to cap it.

## Setup

1. Create the folders:
   ```bash
   sudo mkdir -p /opt/stacks/homarr /opt/appdata/homarr
   sudo chown -R $USER: /opt/stacks/homarr
   ```
2. Add the variables above to `/opt/stacks/_shared/.env` and `/opt/stacks/_shared/.secret` (`chmod 600 .secret`).
3. Save the compose file as `/opt/stacks/homarr/compose.yaml`.
4. Check the configuration, then start the stack:
   ```bash
   cd /opt/stacks/homarr
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret config --quiet
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
   ```
5. Watch the first start: `docker compose logs -f`.

6. Open `http://<SERVER_IP>:7575` and complete the first-run wizard (create the admin user).
7. Add your apps as shortcuts. Use the public URL you already open in a browser, not an internal hostname, if this dashboard will be shared.
8. If Homarr is reachable beyond your LAN, put it behind a reverse proxy with authentication. See [oauth2-proxy](/oauth2-proxy).

> Tip: create a small wrapper such as `alias dc='docker compose --env-file /opt/stacks/_shared/.env --env-file /opt/stacks/_shared/.secret'` so every stack uses the same env files.
{.is-success}
## Updating

```bash
cd /opt/stacks/homarr
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret pull
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
docker image prune -f
```

Back up `/opt/appdata/homarr` before major upgrades. [Watchtower](/watchtower) can automate updates for stateless apps.

## Troubleshooting

- Check status and logs: `docker compose ps` and `docker compose logs --tail=100`.
- **"SECRET_ENCRYPTION_KEY missing" or unreadable secrets:** the key must be a 64-character hex string and must stay the same across upgrades.
- **Board empty after a recreate:** confirm `/opt/appdata/homarr/appdata` is mounted and was not started with an empty new directory.
- **A variable is empty or reported missing (`:?err`):** run the `config` command above to see which variable is not set in your env files.

## Related

- [Docker Stack Catalog](/docker-stacks)
- [Dockge](/dockge)
- [oauth2-proxy](/oauth2-proxy)

---
*Educational example. Review every path, port and credential before you deploy, and never publish your `.secret` file.*
