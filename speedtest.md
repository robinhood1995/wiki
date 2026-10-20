---
title: Speedtest Tracker
description: Speedtest Tracker scheduled internet speed tests stored in MySQL, with Docker Compose.
published: true
date: 2026-10-18T03:28:26.329Z
tags: docker, docker-compose, homelab, monitoring, mysql, network, speedtest
editor: markdown
dateCreated: 2026-09-08T21:15:35.866Z
---

# Speedtest Tracker

> Speedtest Tracker scheduled internet speed tests stored in MySQL, with Docker Compose.
{.is-info}

_Last updated: Oct 2026_

## Overview

Speedtest Tracker runs Ookla speed tests on a schedule (hourly in this example) and charts download, upload, ping and jitter over time. Results can be sent as notifications.

**Why you'd want it:** Hard evidence when your ISP is not delivering what you pay for.

| Item | Value |
|---|---|
| Stack folder | `/opt/stacks/speedtest` |
| Category | Monitoring |
| Main image | `lscr.io/linuxserver/speedtest-tracker:latest` |
| Services | 1 |
| Web UI | http://<SERVER_IP>:30011 |
| Official docs | https://docs.speedtest-tracker.dev/ |

## Requirements

- A Linux host with Docker Engine and the Docker Compose plugin (see [Docker](/docker))
- The shared folder layout and `.env`/`.secret` files from [Docker Folders](/dockerfolders) and [Environment](/environment)
- MySQL/MariaDB with a `speedtest` database (or switch `DB_CONNECTION` to sqlite)
- An app key: `echo -n base64:; openssl rand -base64 32`

## Compose

Folder layout:

```text
/opt/stacks/
├── _shared/
│   ├── .env        # shared, non-secret settings
│   └── .secret     # passwords, tokens and keys (chmod 600, never commit)
└── speedtest/
    ├── compose.yaml
    └── compose.sh
/opt/appdata/speedtest/   # persistent application data
```

`/opt/stacks/speedtest/compose.yaml`:

```yaml
---
services:
  speedtest-tracker:
    image: lscr.io/linuxserver/speedtest-tracker:latest
    container_name: speedtest
    hostname: speedtest
    cap_add:
      - NET_RAW
    env_file:
      - /opt/stacks/_shared/.env
      - /opt/stacks/_shared/.secret
    environment:
      APP_KEY: ${ST_APP_KEY}
      DB_CONNECTION: mysql
      SPEEDTEST_SCHEDULE: "0 * * * *"
      DB_HOST: ${ST_DB_HOST}
      DB_PORT: ${ST_DB_PORT}
      DB_DATABASE: ${ST_DB_DATABASE}
      DB_USERNAME: ${ST_DB_USERNAME}
      DB_PASSWORD: ${ST_DB_PASSWORD}
      APP_DEBUG: "false"
    logging:
      driver: json-file
      options:
        max-file: ${DOCKERLOGGING_MAXFILE:-10}  # Max number of log files
        max-size: ${DOCKERLOGGING_MAXSIZE:-200k}  # Max file size
    volumes:
      - /opt/appdata/speedtest/config:/config
    ports:
      - 30011:80 # Map host port 8080 to container port 80
    restart: unless-stopped
```

`/opt/stacks/speedtest/compose.sh`:

```bash
docker compose \
  --env-file /opt/stacks/_shared/.env \
  --env-file /opt/stacks/_shared/.secret \
  up -d
```

## Environment variables

Compose reads these from the shared files with `--env-file`. Replace every `<PLACEHOLDER>` with your own value.

`/opt/stacks/_shared/.env`:

```ini
PUID=1000
PGID=1000
TZ=<TZ>
ST_DB_HOST=<DB_HOST>
ST_DB_PORT=3306
ST_DB_DATABASE=speedtest
DOCKERLOGGING_MAXFILE=10
DOCKERLOGGING_MAXSIZE=200k
```

`/opt/stacks/_shared/.secret`:

```ini
ST_APP_KEY=<APP_KEY>
ST_DB_USERNAME=<DB_USER>
ST_DB_PASSWORD=<DB_PASSWORD>
```

| Variable | File | Purpose | Example |
|---|---|---|---|
| `PUID` | `.env` | User ID the container runs as (see Docker Folders) | `1000` |
| `PGID` | `.env` | Group ID the container runs as | `1000` |
| `TZ` | `.env` | Time zone in tz database format, e.g. America/New_York | `<TZ>` |
| `ST_APP_KEY` | `.secret` | Speedtest Tracker Laravel app key (base64:...) | `<APP_KEY>` |
| `ST_DB_HOST` | `.env` | MySQL/MariaDB host | `<DB_HOST>` |
| `ST_DB_PORT` | `.env` | MySQL/MariaDB port | `3306` |
| `ST_DB_DATABASE` | `.env` | Database name | `speedtest` |
| `ST_DB_USERNAME` | `.secret` | Database user | `<DB_USER>` |
| `ST_DB_PASSWORD` | `.secret` | Database password | `<DB_PASSWORD>` |
| `DOCKERLOGGING_MAXFILE` | `.env` | Number of rotated container log files to keep | `10` |
| `DOCKERLOGGING_MAXSIZE` | `.env` | Maximum size of each container log file | `200k` |

## Ports

| Host port | Container port | Protocol | Service | Purpose |
|---|---|---|---|---|
| 30011 | 80 | tcp | `speedtest-tracker` | Map host port 8080 to container port 80 |

## Volumes

| Host path | Container path | Service | Purpose |
|---|---|---|---|
| `/opt/appdata/speedtest/config` | `/config` | `speedtest-tracker` | Application configuration |

## Resource limits

No limits are set. This app is light, so add `deploy.resources.limits` only if you need to cap it.

## Setup

1. Create the folders:
   ```bash
   sudo mkdir -p /opt/stacks/speedtest /opt/appdata/speedtest
   sudo chown -R $USER: /opt/stacks/speedtest
   ```
2. Add the variables above to `/opt/stacks/_shared/.env` and `/opt/stacks/_shared/.secret` (`chmod 600 .secret`).
3. Save the compose file as `/opt/stacks/speedtest/compose.yaml` together with the config files shown above.
4. Check the configuration, then start the stack:
   ```bash
   cd /opt/stacks/speedtest
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret config --quiet
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
   ```
5. Watch the first start: `docker compose logs -f`.
6. Generate `ST_APP_KEY` and add the database settings to `.env`/`.secret`.
7. Start the stack (or run `./compose.sh`) and open `http://<SERVER_IP>:30011`.
8. Sign in with the default admin account shown in the docs and change the password.
9. Adjust the schedule with `SPEEDTEST_SCHEDULE` (cron format).

> Tip: create a small wrapper such as `alias dc='docker compose --env-file /opt/stacks/_shared/.env --env-file /opt/stacks/_shared/.secret'` so every stack uses the same env files.
{.is-success}

## Updating

```bash
cd /opt/stacks/speedtest
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret pull
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
docker image prune -f
```

Back up `/opt/appdata/speedtest` before major upgrades. [Watchtower](/watchtower) can automate updates for stateless apps.

## Troubleshooting

- Check status and logs: `docker compose ps` and `docker compose logs --tail=100`.
- **500 error on first load:** `APP_KEY` is missing or not prefixed with `base64:`.
- **Database connection refused:** check `ST_DB_HOST` and that the user may connect from the Docker host.
- **A variable is empty or reported missing (`:?err`):** run the `config` command above to see which variable is not set in your env files.

## Related

- [Docker Stack Catalog](/docker-stacks)
- [Grafana](/grafana)
- [Prometheus](/prometheus)

---
*Educational example. Review every path, port and credential before you deploy, and never publish your `.secret` file.*
