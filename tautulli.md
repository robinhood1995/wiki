---
title: Tautulli
description: Tautulli Plex monitoring, watch history and statistics (LinuxServer image), with Docker Compose.
published: true
date: 2026-10-20T20:01:47.361Z
tags: docker, docker-compose, homelab, media, monitoring, plex, statistics
editor: markdown
dateCreated: 2026-09-08T21:15:35.866Z
---

# Tautulli

> Tautulli Plex monitoring, watch history and statistics (LinuxServer image), with Docker Compose.
{.is-info}

_Last updated: Oct 2026_

## Overview

Tautulli watches your Plex Media Server and records who watched what, when and how (direct play or transcode). It shows live activity, history, graphs and library statistics, and can send notifications such as "new movie added" or "stream started" to Discord, email and more.

**Why you'd want it:** Know how your Plex server is really used, and spot transcoding or bandwidth problems early.

| Item | Value |
|---|---|
| Stack folder | `/opt/stacks/tautulli` |
| Category | Media |
| Main image | `lscr.io/linuxserver/tautulli:latest` |
| Services | 1 |
| Web UI | http://<SERVER_IP>:8182 |
| Official docs | https://docs.linuxserver.io/images/docker-tautulli/ |

## Requirements

- A Linux host with Docker Engine and the Docker Compose plugin (see [Docker](/docker))
- The shared folder layout and `.env`/`.secret` files from [Docker Folders](/dockerfolders) and [Environment](/environment)
- A [Plex](/plex) server and its Plex account

## Compose

Folder layout:

```text
/opt/stacks/
├── _shared/
│   ├── .env        # shared, non-secret settings
│   └── .secret     # passwords, tokens and keys (chmod 600, never commit)
└── tautulli/
    └── compose.yaml
/opt/appdata/tautulli/   # persistent application data
```

`/opt/stacks/tautulli/compose.yaml`:

```yaml
---
services:
  tautulli:
    image: lscr.io/linuxserver/tautulli:latest
    container_name: tautulli
    hostname: tautulli
    env_file:
      - /opt/stacks/_shared/.env
      - /opt/stacks/_shared/.secret
    logging:
      driver: json-file
      options:
        max-file: ${DOCKERLOGGING_MAXFILE:-10}  # Max number of log files
        max-size: ${DOCKERLOGGING_MAXSIZE:-200k}  # Max file size
    volumes:
      - /opt/appdata/tautulli/config:/config # Tautulli settings and database
      - /opt/appdata/tautulli/plex/logs:/plex_logs:ro # optional: Plex logs for the log viewer
    ports:
      - 8182:8181
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
| 8182 | 8181 | tcp | `tautulli` | Web UI |

## Volumes

| Host path | Container path | Service | Purpose |
|---|---|---|---|
| `/opt/appdata/tautulli/config` | `/config` | `tautulli` | Tautulli settings and database |
| `/opt/appdata/tautulli/plex/logs` | `/plex_logs` | `tautulli` | optional: Plex logs for the log viewer (read-only) |

## Resource limits

No limits are set. This app is light, so add `deploy.resources.limits` only if you need to cap it.

## Setup

1. Create the folders:
   ```bash
   sudo mkdir -p /opt/stacks/tautulli /opt/appdata/tautulli
   sudo chown -R $USER: /opt/stacks/tautulli
   ```
2. Add the variables above to `/opt/stacks/_shared/.env` and `/opt/stacks/_shared/.secret` (`chmod 600 .secret`).
3. Save the compose file as `/opt/stacks/tautulli/compose.yaml`.
4. Check the configuration, then start the stack:
   ```bash
   cd /opt/stacks/tautulli
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret config --quiet
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
   ```
5. Watch the first start: `docker compose logs -f`.
6. Open `http://<SERVER_IP>:8182` and run the setup wizard.
7. Sign in with your Plex account, then pick your server (use `<SERVER_IP>` and port 32400).
8. Optional: mount the Plex `Logs` folder at `/plex_logs` and set that path under **Settings > Plex Media Server > Logs Folder** to use the log viewer.
9. Set up notification agents under **Settings > Notification Agents**.

> Tip: create a small wrapper such as `alias dc='docker compose --env-file /opt/stacks/_shared/.env --env-file /opt/stacks/_shared/.secret'` so every stack uses the same env files.
{.is-success}

## Updating

```bash
cd /opt/stacks/tautulli
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret pull
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
docker image prune -f
```

Back up `/opt/appdata/tautulli` before major upgrades. [Watchtower](/watchtower) can automate updates for stateless apps.

## Troubleshooting

- Check status and logs: `docker compose ps` and `docker compose logs --tail=100`.
- **Cannot connect to Plex:** use the server IP rather than `localhost`, and enable **Use SSL** only if Plex requires secure connections.
- **Port 8181 in use:** this example publishes it on 8182; change the host side if needed.
- **History missing after reinstall:** restore `/config/tautulli.db` from a backup.
- **A variable is empty or reported missing (`:?err`):** run the `config` command above to see which variable is not set in your env files.

## Related

- [Docker Stack Catalog](/docker-stacks)
- [Plex](/plex)
- [Ombi](/ombi)
- [Seerr](/seerr)
- [Dedupe](/dedupe)

---
*Educational example. Review every path, port and credential before you deploy, and never publish your `.secret` file.*
