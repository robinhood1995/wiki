---
title: Ombi
description: Ombi media request portal for Plex, Jellyfin and Emby users (LinuxServer image), with Docker Compose.
published: true
date: 2026-10-20T20:01:39.417Z
tags: docker, docker-compose, homelab, jellyfin, media, plex, requests
editor: markdown
dateCreated: 2026-09-08T21:15:35.866Z
---

# Ombi

> Ombi media request portal for Plex, Jellyfin and Emby users (LinuxServer image), with Docker Compose.
{.is-info}

_Last updated: Oct 2026_

## Overview

Ombi lets the users of your Plex, Jellyfin or Emby server request movies, TV shows and music. Requests can be auto-approved or reviewed, then sent to Sonarr, Radarr and Lidarr. Users sign in with their media server account.

**Why you'd want it:** Stop collecting requests by text message; let users ask, track and get notified on their own.

| Item | Value |
|---|---|
| Stack folder | `/opt/stacks/ombi` |
| Category | Media |
| Main image | `lscr.io/linuxserver/ombi:latest` |
| Services | 1 |
| Web UI | http://<SERVER_IP>:3579 |
| Official docs | https://docs.ombi.app/ |

## Requirements

- A Linux host with Docker Engine and the Docker Compose plugin (see [Docker](/docker))
- The shared folder layout and `.env`/`.secret` files from [Docker Folders](/dockerfolders) and [Environment](/environment)
- A Plex, Jellyfin or Emby server
- Sonarr/Radarr for automatic downloads (see [Arr Stack](/arr-stack))

## Compose

Folder layout:

```text
/opt/stacks/
├── _shared/
│   ├── .env        # shared, non-secret settings
│   └── .secret     # passwords, tokens and keys (chmod 600, never commit)
└── ombi/
    └── compose.yaml
/opt/appdata/ombi/   # persistent application data
```

`/opt/stacks/ombi/compose.yaml`:

```yaml
services:
  ombi:
    image: lscr.io/linuxserver/ombi:latest
    container_name: ombi
    hostname: ombi
    env_file:
      - /opt/stacks/_shared/.env
      - /opt/stacks/_shared/.secret
    volumes:
      - /opt/appdata/ombi/config:/config
    logging:
      driver: json-file
      options:
        max-file: ${DOCKERLOGGING_MAXFILE:-10}  # Max number of log files
        max-size: ${DOCKERLOGGING_MAXSIZE:-200k}  # Max file size
    ports:
      - 3579:3579
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
| 3579 | 3579 | tcp | `ombi` | Web UI |

## Volumes

| Host path | Container path | Service | Purpose |
|---|---|---|---|
| `/opt/appdata/ombi/config` | `/config` | `ombi` | Application configuration |

## Resource limits

No limits are set. This app is light, so add `deploy.resources.limits` only if you need to cap it.

## Setup

1. Create the folders:
   ```bash
   sudo mkdir -p /opt/stacks/ombi /opt/appdata/ombi
   sudo chown -R $USER: /opt/stacks/ombi
   ```
2. Add the variables above to `/opt/stacks/_shared/.env` and `/opt/stacks/_shared/.secret` (`chmod 600 .secret`).
3. Save the compose file as `/opt/stacks/ombi/compose.yaml`.
4. Check the configuration, then start the stack:
   ```bash
   cd /opt/stacks/ombi
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret config --quiet
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
   ```
5. Watch the first start: `docker compose logs -f`.
6. Open `http://<SERVER_IP>:3579` and follow the wizard: connect your media server and create the admin user.
7. Under **Settings > Media Server**, test the connection and run a library sync.
8. Under **Settings > TV / Movies**, add Sonarr and Radarr with their API keys and quality profiles.
9. Import users from your media server and set their request limits.

> Tip: create a small wrapper such as `alias dc='docker compose --env-file /opt/stacks/_shared/.env --env-file /opt/stacks/_shared/.secret'` so every stack uses the same env files.
{.is-success}

## Updating

```bash
cd /opt/stacks/ombi
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret pull
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
docker image prune -f
```

Back up `/opt/appdata/ombi` before major upgrades. [Watchtower](/watchtower) can automate updates for stateless apps.

## Troubleshooting

- Check status and logs: `docker compose ps` and `docker compose logs --tail=100`.
- **Choosing between Ombi and Seerr:** both do the same job; [Seerr](/seerr) is the more active project today. Run one, not both.
- **Library not showing as available:** run a full sync under Settings > Media Server.
- **Database locked errors:** keep `/config` on local storage, not a network share.
- **A variable is empty or reported missing (`:?err`):** run the `config` command above to see which variable is not set in your env files.

## Related

- [Docker Stack Catalog](/docker-stacks)
- [Seerr](/seerr)
- [Arr Stack](/arr-stack)
- [Plex](/plex)
- [Jellyfin](/jellyfin)
- [Tautulli](/tautulli)

---
*Educational example. Review every path, port and credential before you deploy, and never publish your `.secret` file.*
