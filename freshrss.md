---
title: FreshRSS
description: FreshRSS self-hosted RSS and news feed reader with Docker Compose.
published: true
date: 2026-10-20T20:01:20.832Z
tags: docker, docker-compose, homelab, productivity, reading, news, rss
editor: markdown
dateCreated: 2026-09-08T21:15:35.866Z
---

# FreshRSS

> FreshRSS self-hosted RSS and news feed reader with Docker Compose.
{.is-info}

_Last updated: Oct 2026_

## Overview

FreshRSS is a fast, multi-user RSS aggregator. Subscribe to blogs, news sites, YouTube channels and forums, and read everything in one place. It supports the Google Reader and Fever APIs, so mobile apps such as Readrops, FeedMe or Reeder can sync with it.

**Why you'd want it:** Follow the sites you care about without algorithms or ads, synced across all your devices.

| Item | Value |
|---|---|
| Stack folder | `/opt/stacks/freshrss` |
| Category | Productivity and reading |
| Main image | `freshrss/freshrss:latest` |
| Services | 1 |
| Web UI | http://<SERVER_IP>:30200 |
| Official docs | https://freshrss.github.io/FreshRSS/en/admins/01_Index.html |

## Requirements

- A Linux host with Docker Engine and the Docker Compose plugin (see [Docker](/docker))
- The shared folder layout and `.env`/`.secret` files from [Docker Folders](/dockerfolders) and [Environment](/environment)
- Optional: MySQL/MariaDB or PostgreSQL (SQLite is fine for a household)

## Compose

Folder layout:

```text
/opt/stacks/
├── _shared/
│   ├── .env        # shared, non-secret settings
│   └── .secret     # passwords, tokens and keys (chmod 600, never commit)
└── freshrss/
    └── compose.yaml
/opt/appdata/freshrss/   # persistent application data
```

`/opt/stacks/freshrss/compose.yaml`:

```yaml
---
services:
  freshrss:
    image: freshrss/freshrss:latest
    container_name: freshrss
    hostname: freshrss
    env_file:
      - /opt/stacks/_shared/.env
      - /opt/stacks/_shared/.secret
    environment:
      - CRON_MIN=5 # refresh feeds at minute 5 of every hour
    logging:
      driver: json-file
      options:
        max-file: ${DOCKERLOGGING_MAXFILE:-10}  # Max number of log files
        max-size: ${DOCKERLOGGING_MAXSIZE:-200k}  # Max file size
    volumes:
      - /opt/appdata/freshrss/freshrss_data:/var/www/FreshRSS/data
      - /opt/appdata/freshrss/freshrss_extensions:/var/www/FreshRSS/extensions
    ports:
      - 30200:80 # web UI
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
| 30200 | 80 | tcp | `freshrss` | web UI |

## Volumes

| Host path | Container path | Service | Purpose |
|---|---|---|---|
| `/opt/appdata/freshrss/freshrss_data` | `/var/www/FreshRSS/data` | `freshrss` | Feeds, users and SQLite database |
| `/opt/appdata/freshrss/freshrss_extensions` | `/var/www/FreshRSS/extensions` | `freshrss` | Third-party extensions |

## Resource limits

No limits are set. This app is light, so add `deploy.resources.limits` only if you need to cap it.

## Setup

1. Create the folders:
   ```bash
   sudo mkdir -p /opt/stacks/freshrss /opt/appdata/freshrss
   sudo chown -R $USER: /opt/stacks/freshrss
   ```
2. Add the variables above to `/opt/stacks/_shared/.env` and `/opt/stacks/_shared/.secret` (`chmod 600 .secret`).
3. Save the compose file as `/opt/stacks/freshrss/compose.yaml`.
4. Check the configuration, then start the stack:
   ```bash
   cd /opt/stacks/freshrss
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret config --quiet
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
   ```
5. Watch the first start: `docker compose logs -f`.
6. Open `http://<SERVER_IP>:30200` and complete the installer: choose SQLite (or your database) and create the admin user.
7. Add feeds, or import an OPML file from your old reader.
8. For mobile apps, enable **Settings > Authentication > Allow API access**, then set an API password in your profile.

> Tip: create a small wrapper such as `alias dc='docker compose --env-file /opt/stacks/_shared/.env --env-file /opt/stacks/_shared/.secret'` so every stack uses the same env files.
{.is-success}

## Updating

```bash
cd /opt/stacks/freshrss
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret pull
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
docker image prune -f
```

Back up `/opt/appdata/freshrss` before major upgrades. [Watchtower](/watchtower) can automate updates for stateless apps.

## Troubleshooting

- Check status and logs: `docker compose ps` and `docker compose logs --tail=100`.
- **Feeds not refreshing:** `CRON_MIN` sets the minutes when the built-in cron runs; check `docker compose logs` for cron output.
- **Wrong links behind a reverse proxy:** set the base URL under **Administration > System configuration**.
- **Extensions not loading:** put them in the `extensions` volume and enable them in the UI.
- **A variable is empty or reported missing (`:?err`):** run the `config` command above to see which variable is not set in your env files.

## Related

- [Docker Stack Catalog](/docker-stacks)
- [13ft](/13ft)

---
*Educational example. Review every path, port and credential before you deploy, and never publish your `.secret` file.*
