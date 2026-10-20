---
title: Nebula Sync (Pi-hole)
description: nebula-sync keeps two or more Pi-hole v6 servers in sync (settings, ad lists, domains), with Docker Compose.
published: true
date: 2026-10-20T20:03:19.665Z
tags: docker, docker-compose, homelab, stack-management, adblock, dns, pihole, sync
editor: markdown
dateCreated: 2026-09-08T21:26:12.838Z
---

# Nebula Sync (Pi-hole)

> nebula-sync keeps two or more Pi-hole v6 servers in sync (settings, ad lists, domains), with Docker Compose.
{.is-info}

_Last updated: Oct 2026_

## Overview

nebula-sync copies configuration from a primary Pi-hole to one or more replicas through the Pi-hole v6 API. With `FULL_SYNC=true` it uses Teleporter to copy everything; otherwise you choose what to sync. It runs on a cron schedule, hourly in this example.

**Why you'd want it:** Run two Pi-holes for redundant DNS and only ever change settings on one of them.

| Item | Value |
|---|---|
| Stack folder | `/opt/stacks/nebula` |
| Category | Stack management |
| Main image | `ghcr.io/lovelaze/nebula-sync:latest` |
| Services | 1 |
| Web UI | None (runs on a schedule; check the logs) |
| Official docs | https://github.com/lovelaze/nebula-sync |

## Requirements

- A Linux host with Docker Engine and the Docker Compose plugin (see [Docker](/docker))
- The shared folder layout and `.env`/`.secret` files from [Docker Folders](/dockerfolders) and [Environment](/environment)
- Two or more Pi-hole **v6** servers
- An app password for each Pi-hole (Settings > Web interface / API > Configure app password)

## Compose

Folder layout:

```text
/opt/stacks/
├── _shared/
│   ├── .env        # shared, non-secret settings
│   └── .secret     # passwords, tokens and keys (chmod 600, never commit)
└── nebula/
    └── compose.yaml
/opt/appdata/nebula/   # persistent application data
```

`/opt/stacks/nebula/compose.yaml`:

```yaml
---
services:
  nebula-sync:
    image: ghcr.io/lovelaze/nebula-sync:latest
    container_name: nebula
    hostname: nebula
    env_file:
      - /opt/stacks/_shared/.env
      - /opt/stacks/_shared/.secret
    environment:
      # PRIMARY and REPLICAS come from _shared/.secret via env_file
      - FULL_SYNC=true
      - RUN_GRAVITY=true
      - SYNC_GRAVITY_DOMAIN_LIST=true
      - CRON=0 * * * *
    logging:
      driver: json-file
      options:
        max-file: ${DOCKERLOGGING_MAXFILE:-10}  # Max number of log files
        max-size: ${DOCKERLOGGING_MAXSIZE:-200k}  # Max file size
    volumes:
      - /opt/appdata/nebula/data:/app/data
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

`/opt/stacks/_shared/.secret`:

```ini
PRIMARY=http://<PIHOLE1_IP>|<PIHOLE1_APP_PASSWORD>
REPLICAS=http://<PIHOLE2_IP>|<PIHOLE2_APP_PASSWORD>
```

| Variable | File | Purpose | Example |
|---|---|---|---|
| `PUID` | `.env` | User ID the container runs as (see Docker Folders) | `1000` |
| `PGID` | `.env` | Group ID the container runs as | `1000` |
| `TZ` | `.env` | Time zone in tz database format, e.g. America/New_York | `<TZ>` |
| `DOCKERLOGGING_MAXFILE` | `.env` | Number of rotated container log files to keep | `10` |
| `DOCKERLOGGING_MAXSIZE` | `.env` | Maximum size of each container log file | `200k` |
| `PRIMARY` | `.secret` | nebula-sync: primary Pi-hole as URL\|app-password | see the `.secret` block above |
| `REPLICAS` | `.secret` | nebula-sync: comma-separated replicas as URL\|app-password | see the `.secret` block above |

## Ports

This stack publishes no ports.

## Volumes

| Host path | Container path | Service | Purpose |
|---|---|---|---|
| `/opt/appdata/nebula/data` | `/app/data` | `nebula-sync` | nebula-sync working data |

## Resource limits

No limits are set. This app is light, so add `deploy.resources.limits` only if you need to cap it.

## Setup

1. Create the folders:
   ```bash
   sudo mkdir -p /opt/stacks/nebula /opt/appdata/nebula
   sudo chown -R $USER: /opt/stacks/nebula
   ```
2. Add the variables above to `/opt/stacks/_shared/.env` and `/opt/stacks/_shared/.secret` (`chmod 600 .secret`).
3. Save the compose file as `/opt/stacks/nebula/compose.yaml`.
4. Put `PRIMARY` and `REPLICAS` in `.secret` as shown above (`URL|password`, replicas comma-separated).
5. Check the configuration, then start the stack:
   ```bash
   cd /opt/stacks/nebula
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret config --quiet
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
   ```
6. Watch the first start: `docker compose logs -f`.
7. Watch `docker compose logs -f`; the first sync runs at the next cron time.
8. Make a small change on the primary and confirm it appears on the replicas after the next run.

> Tip: create a small wrapper such as `alias dc='docker compose --env-file /opt/stacks/_shared/.env --env-file /opt/stacks/_shared/.secret'` so every stack uses the same env files.
{.is-success}

## Updating

```bash
cd /opt/stacks/nebula
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret pull
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
docker image prune -f
```

Back up `/opt/appdata/nebula` before major upgrades. [Watchtower](/watchtower) can automate updates for stateless apps.

## Troubleshooting

- Check status and logs: `docker compose ps` and `docker compose logs --tail=100`.
- **401 or authentication errors:** use the Pi-hole app password, not the web login password, and include `http://` or `https://`.
- **TLS errors with self-signed certificates:** set `CLIENT_SKIP_TLS_VERIFICATION=true`, or use `http://` on your LAN.
- **Pi-hole v5:** not supported; use Gravity Sync or upgrade to v6.
- **Replica-only settings overwritten:** with `FULL_SYNC=true` everything is copied; set it to false and pick `SYNC_*` options instead.
- **A variable is empty or reported missing (`:?err`):** run the `config` command above to see which variable is not set in your env files.

## Related

- [Docker Stack Catalog](/docker-stacks)
- [Mailrise](/mailrise)
- [Watchtower](/watchtower)

---
*Educational example. Review every path, port and credential before you deploy, and never publish your `.secret` file.*
