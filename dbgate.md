---
title: DbGate
description: DbGate web database client for MySQL, PostgreSQL, SQL Server, MongoDB, Redis and SQLite, with Docker Compose.
published: true
date: 2026-10-20T20:01:04.953Z
tags: admin, database, developer, docker, docker-compose, homelab, mysql, postgres, mongodb
editor: markdown
dateCreated: 2026-09-08T21:26:12.838Z
---

# DbGate

> DbGate web database client for MySQL, PostgreSQL, SQL Server, MongoDB, Redis and SQLite, with Docker Compose.
{.is-info}

_Last updated: Oct 2026_

## Overview

DbGate is a modern database client that runs in the browser. It supports SQL and NoSQL databases, has a data grid editor, query designer, ER diagrams, and import/export to CSV, Excel and JSON. Saved connections and scripts are kept in `/app/data`.

**Why you'd want it:** A friendlier, more powerful alternative to Adminer when you work with several databases regularly.

| Item | Value |
|---|---|
| Stack folder | `/opt/stacks/dbgate` |
| Category | Developer and IT tools |
| Main image | `dbgate/dbgate:latest` |
| Services | 1 |
| Web UI | http://<SERVER_IP>:30020 |
| Official docs | https://docs.dbgate.io/ |

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
└── dbgate/
    └── compose.yaml
/opt/appdata/dbgate/   # persistent application data
```

`/opt/stacks/dbgate/compose.yaml`:

```yaml
---
services:
  dbgate:
    image: dbgate/dbgate:latest
    container_name: dbgate
    hostname: dbgate
    env_file:
      - /opt/stacks/_shared/.env
      - /opt/stacks/_shared/.secret
    # Optional: Set credentials for accessing the web app.
    # environment:
    #   - LOGIN=<USERNAME>
    #   - PASSWORD=<STRONG_PASSWORD>
    logging:
      driver: json-file
      options:
        max-file: ${DOCKERLOGGING_MAXFILE:-10}  # Max number of log files
        max-size: ${DOCKERLOGGING_MAXSIZE:-200k}  # Max file size
    ports:
      - 30020:3000
    volumes:
      # This line is crucial for persisting your connections, scripts, and archives.
      - /opt/appdata/dbgate/dbgate_data:/app/data
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
| 30020 | 3000 | tcp | `dbgate` | Web UI |

## Volumes

| Host path | Container path | Service | Purpose |
|---|---|---|---|
| `/opt/appdata/dbgate/dbgate_data` | `/app/data` | `dbgate` | Saved connections, scripts and archives |

## Resource limits

No limits are set. This app is light, so add `deploy.resources.limits` only if you need to cap it.

## Setup

1. Create the folders:
   ```bash
   sudo mkdir -p /opt/stacks/dbgate /opt/appdata/dbgate
   sudo chown -R $USER: /opt/stacks/dbgate
   ```
2. Add the variables above to `/opt/stacks/_shared/.env` and `/opt/stacks/_shared/.secret` (`chmod 600 .secret`).
3. Save the compose file as `/opt/stacks/dbgate/compose.yaml`.
4. Check the configuration, then start the stack:
   ```bash
   cd /opt/stacks/dbgate
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret config --quiet
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
   ```
5. Watch the first start: `docker compose logs -f`.
6. Open `http://<SERVER_IP>:30020` and click **New connection**.
7. Pick the engine, enter `<DB_HOST>`, port, user and password, then **Test** and **Save**.
8. Before using it beyond your own desk, enable the login: uncomment `LOGIN`/`PASSWORD` in the compose file (keep the real password in `.secret`).

> Tip: create a small wrapper such as `alias dc='docker compose --env-file /opt/stacks/_shared/.env --env-file /opt/stacks/_shared/.secret'` so every stack uses the same env files.
{.is-success}

## Updating

```bash
cd /opt/stacks/dbgate
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret pull
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
docker image prune -f
```

Back up `/opt/appdata/dbgate` before major upgrades. [Watchtower](/watchtower) can automate updates for stateless apps.

## Troubleshooting

- Check status and logs: `docker compose ps` and `docker compose logs --tail=100`.
- **Anyone on the LAN can open it:** with no `LOGIN`/`PASSWORD` set, DbGate has no authentication.
- **Connections lost after an update:** `/app/data` must be on persistent storage.
- **Connection timeout:** check firewalls between the Docker host and the database, and the bind address of the database server.
- **A variable is empty or reported missing (`:?err`):** run the `config` command above to see which variable is not set in your env files.

## Related

- [Docker Stack Catalog](/docker-stacks)
- [Adminer](/adminer)
- [IT-Tools](/it-tools)

---
*Educational example. Review every path, port and credential before you deploy, and never publish your `.secret` file.*
