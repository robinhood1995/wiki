---
title: ArchiSteamFarm
description: ArchiSteamFarm (ASF) Steam trading-card farming bot with web IPC UI, using Docker Compose.
published: true
date: 2026-10-20T20:01:02.265Z
tags: docker, docker-compose, homelab, bot, gaming, steam
editor: markdown
dateCreated: 2026-09-08T21:26:12.838Z
---

# ArchiSteamFarm

> ArchiSteamFarm (ASF) Steam trading-card farming bot with web IPC UI, using Docker Compose.
{.is-info}

_Last updated: Oct 2026_

## Overview

ArchiSteamFarm (ASF) logs in to one or more Steam accounts and idles your games to collect the trading cards you are entitled to, without you running Steam. It also handles redeeming keys and other account chores. The optional IPC web UI (ASF-ui) lets you manage bots from a browser.

**Why you'd want it:** Collect Steam trading cards automatically on an always-on server instead of leaving a gaming PC running.

| Item | Value |
|---|---|
| Stack folder | `/opt/stacks/archisteamfarm` |
| Category | Gaming |
| Main image | `justarchi/archisteamfarm:released` |
| Services | 1 |
| Web UI | http://<SERVER_IP>:30019 (IPC, optional) |
| Official docs | https://github.com/JustArchiNET/ArchiSteamFarm/wiki/Docker |

## Requirements

- A Linux host with Docker Engine and the Docker Compose plugin (see [Docker](/docker))
- The shared folder layout and `.env`/`.secret` files from [Docker Folders](/dockerfolders) and [Environment](/environment)
- A Steam account; Steam Guard mobile authenticator recommended

## Compose

Folder layout:

```text
/opt/stacks/
├── _shared/
│   ├── .env        # shared, non-secret settings
│   └── .secret     # passwords, tokens and keys (chmod 600, never commit)
└── archisteamfarm/
    └── compose.yaml
/opt/appdata/archisteamfarm/   # persistent application data
```

`/opt/stacks/archisteamfarm/compose.yaml`:

```yaml
---
services:
  asf:
    image: justarchi/archisteamfarm:released
    container_name: archisteamfarm
    hostname: archisteamfarm
    env_file:
      - /opt/stacks/_shared/.env
      - /opt/stacks/_shared/.secret
    logging:
      driver: json-file
      options:
        max-file: ${DOCKERLOGGING_MAXFILE:-10}  # Max number of log files
        max-size: ${DOCKERLOGGING_MAXSIZE:-200k}  # Max file size
    volumes:
      - /opt/appdata/archisteamfarm/config:/app/config
      - /opt/appdata/archisteamfarm/logs:/app/logs
    ports:
      - 30019:1242 # Optional: only needed if you use ASF's IPC feature
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
| 30019 | 1242 | tcp | `asf` | Optional: only needed if you use ASF's IPC feature |

## Volumes

| Host path | Container path | Service | Purpose |
|---|---|---|---|
| `/opt/appdata/archisteamfarm/config` | `/app/config` | `asf` | Bot and ASF configuration |
| `/opt/appdata/archisteamfarm/logs` | `/app/logs` | `asf` | ASF logs |

## Resource limits

No limits are set. This app is light, so add `deploy.resources.limits` only if you need to cap it.

## Setup

1. Create the folders:
   ```bash
   sudo mkdir -p /opt/stacks/archisteamfarm /opt/appdata/archisteamfarm
   sudo chown -R $USER: /opt/stacks/archisteamfarm
   ```
2. Add the variables above to `/opt/stacks/_shared/.env` and `/opt/stacks/_shared/.secret` (`chmod 600 .secret`).
3. Save the compose file as `/opt/stacks/archisteamfarm/compose.yaml`.
4. Check the configuration, then start the stack:
   ```bash
   cd /opt/stacks/archisteamfarm
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret config --quiet
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
   ```
5. Watch the first start: `docker compose logs -f`.
6. Start the stack once so ASF creates `/opt/appdata/archisteamfarm/config`.
7. Create a bot file such as `config/MyBot.json` with `{"Enabled": true, "SteamLogin": "<STEAM_USER>", "SteamPassword": "<STEAM_PASSWORD>"}`, or add the bot from the web UI.
8. To use the web UI from your LAN, edit `config/IPC.config` so Kestrel listens on `http://*:1242`, and set an `IPCPassword` in `ASF.json`.
9. Restart the stack and approve the Steam Guard prompt (`docker attach archisteamfarm` lets you type the code, Ctrl+P Ctrl+Q to detach).

> Tip: create a small wrapper such as `alias dc='docker compose --env-file /opt/stacks/_shared/.env --env-file /opt/stacks/_shared/.secret'` so every stack uses the same env files.
{.is-success}

## Updating

```bash
cd /opt/stacks/archisteamfarm
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret pull
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
docker image prune -f
```

Back up `/opt/appdata/archisteamfarm` before major upgrades. [Watchtower](/watchtower) can automate updates for stateless apps.

## Troubleshooting

- Check status and logs: `docker compose ps` and `docker compose logs --tail=100`.
- **Web UI not reachable:** by default IPC only listens on localhost inside the container; change `IPC.config` as above.
- **Bot asks for a 2FA code on every start:** import your mobile authenticator or keep the session files in `/app/config`.
- Bot files contain your Steam password. Keep `/opt/appdata/archisteamfarm` private and out of Git.
- **A variable is empty or reported missing (`:?err`):** run the `config` command above to see which variable is not set in your env files.

## Related

- [Docker Stack Catalog](/docker-stacks)
- [code-server](/code-server)

---
*Educational example. Review every path, port and credential before you deploy, and never publish your `.secret` file.*
