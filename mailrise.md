---
title: Mailrise
description: Mailrise SMTP gateway that turns emails from devices into Apprise push notifications, with Docker Compose.
published: true
date: 2026-10-20T20:01:31.441Z
tags: docker, docker-compose, homelab, stack-management, apprise, notifications, smtp
editor: markdown
dateCreated: 2026-09-08T21:15:35.866Z
---

# Mailrise

> Mailrise SMTP gateway that turns emails from devices into Apprise push notifications, with Docker Compose.
{.is-info}

_Last updated: Oct 2026_

## Overview

Lots of homelab gear can only send alerts by email: UPS units, NAS boxes, printers, routers, cameras. Mailrise is a small SMTP server that receives those emails and forwards them as push notifications through Apprise to Pushover, Discord, Telegram, Gotify, ntfy, Slack and many more.

**Why you'd want it:** Get device alerts on your phone instantly without running a real mail server.

| Item | Value |
|---|---|
| Stack folder | `/opt/stacks/mailrise` |
| Category | Stack management |
| Main image | `yoryan/mailrise:latest` |
| Services | 1 |
| Web UI | None (SMTP on port 8025) |
| Official docs | https://github.com/YoRyan/mailrise |

## Requirements

- A Linux host with Docker Engine and the Docker Compose plugin (see [Docker](/docker))
- The shared folder layout and `.env`/`.secret` files from [Docker Folders](/dockerfolders) and [Environment](/environment)
- An account with a notification service supported by Apprise

## Compose

Folder layout:

```text
/opt/stacks/
├── _shared/
│   ├── .env        # shared, non-secret settings
│   └── .secret     # passwords, tokens and keys (chmod 600, never commit)
└── mailrise/
    └── compose.yaml
/opt/appdata/mailrise/   # persistent application data
```

`/opt/stacks/mailrise/compose.yaml`:

```yaml
---
services:
  mailrise:
    image: yoryan/mailrise:latest
    container_name: mailrise
    hostname: mailrise
    env_file:
      - /opt/stacks/_shared/.env
      - /opt/stacks/_shared/.secret
    logging:
      driver: json-file
      options:
        max-file: ${DOCKERLOGGING_MAXFILE:-10}  # Max number of log files
        max-size: ${DOCKERLOGGING_MAXSIZE:-200k}  # Max file size
    volumes:
      # Mailrise configuration (see below)
      - /opt/appdata/mailrise/mailrise.conf:/etc/mailrise.conf
    ports:
      - 8025:8025 # SMTP listener
    restart: unless-stopped
```

`/opt/appdata/mailrise/mailrise.conf`:

```yaml
# Each key under "configs" becomes an address: <key>@mailrise.xyz
configs:
  discord:                     # send to discord@mailrise.xyz
    urls:
      - discord://<WEBHOOK_ID>/<WEBHOOK_TOKEN>
  pushover:                    # send to pushover@mailrise.xyz
    urls:
      - pover://<PUSHOVER_USER_KEY>@<PUSHOVER_APP_TOKEN>
  ntfy:                        # send to ntfy@mailrise.xyz
    urls:
      - ntfys://<NTFY_HOST>/<TOPIC>
    mailrise:
      title_template: "$subject"
      body_template: "$body"
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
| 8025 | 8025 | tcp | `mailrise` | SMTP listener |

## Volumes

| Host path | Container path | Service | Purpose |
|---|---|---|---|
| `/opt/appdata/mailrise/mailrise.conf` | `/etc/mailrise.conf` | `mailrise` | Mailrise configuration (routes and Apprise URLs) |

## Resource limits

No limits are set. This app is light, so add `deploy.resources.limits` only if you need to cap it.

## Setup

1. Create the folders:
   ```bash
   sudo mkdir -p /opt/stacks/mailrise /opt/appdata/mailrise
   sudo chown -R $USER: /opt/stacks/mailrise
   ```
2. Add the variables above to `/opt/stacks/_shared/.env` and `/opt/stacks/_shared/.secret` (`chmod 600 .secret`).
3. Save the compose file as `/opt/stacks/mailrise/compose.yaml` and create the config file shown above.
4. Create `/opt/appdata/mailrise/mailrise.conf` from the example above with your own Apprise URLs. It must exist before the first start, otherwise Docker creates a folder in its place.
5. Check the configuration, then start the stack:
   ```bash
   cd /opt/stacks/mailrise
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret config --quiet
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
   ```
6. Watch the first start: `docker compose logs -f`.
7. On each device, set the SMTP server to `<SERVER_IP>`, port `8025`, no TLS or authentication, and the recipient to `<CONFIG_NAME>@mailrise.xyz` (for example `discord@mailrise.xyz`).
8. Send a test email from the device and check `docker compose logs`.

> Tip: create a small wrapper such as `alias dc='docker compose --env-file /opt/stacks/_shared/.env --env-file /opt/stacks/_shared/.secret'` so every stack uses the same env files.
{.is-success}

## Updating

```bash
cd /opt/stacks/mailrise
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret pull
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
docker image prune -f
```

Back up `/opt/appdata/mailrise` before major upgrades. [Watchtower](/watchtower) can automate updates for stateless apps.

## Troubleshooting

- Check status and logs: `docker compose ps` and `docker compose logs --tail=100`.
- **Device refuses to send:** some devices insist on port 25 or 587. Map that host port to 8025 instead.
- **"No such config" in the logs:** the part before `@mailrise.xyz` must match a key under `configs:`.
- Mailrise has no authentication by default. Keep port 8025 on your LAN.
- **A variable is empty or reported missing (`:?err`):** run the `config` command above to see which variable is not set in your env files.

## Related

- [Docker Stack Catalog](/docker-stacks)
- [Watchtower](/watchtower)
- [Speedtest Tracker](/speedtest)

---
*Educational example. Review every path, port and credential before you deploy, and never publish your `.secret` file.*
