---
title: Adminer
description: Adminer single-file web UI for MySQL, MariaDB, PostgreSQL, SQLite and more, with Docker Compose.
published: true
date: 2026-10-20T20:00:59.575Z
tags: developer, docker, docker-compose, homelab, mysql, postgres, admin, database
editor: markdown
dateCreated: 2026-09-08T21:15:35.866Z
---

# Adminer

> Adminer single-file web UI for MySQL, MariaDB, PostgreSQL, SQLite and more, with Docker Compose.
{.is-info}

_Last updated: Oct 2026_

## Overview

Adminer is a full database management tool in one small PHP app. Log in to MySQL/MariaDB, PostgreSQL, SQLite, MS SQL or Oracle servers, browse and edit tables, run SQL, and import or export dumps.

**Why you'd want it:** A quick, lightweight way to look inside the databases behind your other stacks without installing a desktop client.

| Item | Value |
|---|---|
| Stack folder | `/opt/stacks/adminer` |
| Category | Developer and IT tools |
| Main image | `adminer:latest` |
| Services | 1 |
| Web UI | http://<SERVER_IP>:30001 |
| Official docs | https://hub.docker.com/_/adminer |

## Requirements

- A Linux host with Docker Engine and the Docker Compose plugin (see [Docker](/docker))
- The shared folder layout and `.env`/`.secret` files from [Docker Folders](/dockerfolders) and [Environment](/environment)
- A database server reachable from the Docker host

## Compose

Folder layout:

```text
/opt/stacks/
├── _shared/
│   ├── .env        # shared, non-secret settings
│   └── .secret     # passwords, tokens and keys (chmod 600, never commit)
└── adminer/
    └── compose.yaml
/opt/appdata/adminer/   # persistent application data
```

`/opt/stacks/adminer/compose.yaml`:

```yaml
---
services:
  adminer:
    image: adminer:latest
    container_name: adminer
    hostname: adminer
    env_file:
      - /opt/stacks/_shared/.env
      - /opt/stacks/_shared/.secret
    logging:
      driver: json-file
      options:
        max-file: ${DOCKERLOGGING_MAXFILE:-10}  # Max number of log files
        max-size: ${DOCKERLOGGING_MAXSIZE:-200k}  # Max file size
    ports:
      - 30001:8080
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
| 30001 | 8080 | tcp | `adminer` | Web UI |

## Volumes

No volumes.

## Resource limits

No limits are set. This app is light, so add `deploy.resources.limits` only if you need to cap it.

## Setup

1. Create the folders:
   ```bash
   sudo mkdir -p /opt/stacks/adminer /opt/appdata/adminer
   sudo chown -R $USER: /opt/stacks/adminer
   ```
2. Add the variables above to `/opt/stacks/_shared/.env` and `/opt/stacks/_shared/.secret` (`chmod 600 .secret`).
3. Save the compose file as `/opt/stacks/adminer/compose.yaml`.
4. Check the configuration, then start the stack:
   ```bash
   cd /opt/stacks/adminer
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret config --quiet
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
   ```
5. Watch the first start: `docker compose logs -f`.
6. Open `http://<SERVER_IP>:30001`.
7. Pick the system (MySQL, PostgreSQL ...), enter the server as `<DB_HOST>:<PORT>`, then the user, password and database.
8. Optional: set `ADMINER_DEFAULT_SERVER=<DB_HOST>` in the compose `environment` to pre-fill the server field.

> Tip: create a small wrapper such as `alias dc='docker compose --env-file /opt/stacks/_shared/.env --env-file /opt/stacks/_shared/.secret'` so every stack uses the same env files.
{.is-success}

## Updating

```bash
cd /opt/stacks/adminer
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret pull
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
docker image prune -f
```

Back up `/opt/appdata/adminer` before major upgrades. [Watchtower](/watchtower) can automate updates for stateless apps.

## Troubleshooting

- Check status and logs: `docker compose ps` and `docker compose logs --tail=100`.
- **Cannot connect to `localhost`:** inside the container, `localhost` is Adminer itself. Use the database host IP or container name.
- **Upload too large on import:** split the dump or import it from the command line.
- Adminer can reach every database it can route to. Do not expose it to the internet.
- **A variable is empty or reported missing (`:?err`):** run the `config` command above to see which variable is not set in your env files.

## Related

- [Docker Stack Catalog](/docker-stacks)
- [DbGate](/dbgate)
- [IT-Tools](/it-tools)

---
*Educational example. Review every path, port and credential before you deploy, and never publish your `.secret` file.*
