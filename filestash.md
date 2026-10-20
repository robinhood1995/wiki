---
title: Filestash
description: Filestash web file manager with Collabora Online for in-browser office editing, using Docker Compose.
published: true
date: 2026-10-18T03:27:25.808Z
tags: docker, docker-compose, documents, homelab, collabora, file-manager, files, office
editor: markdown
dateCreated: 2026-09-08T21:26:12.838Z
---

# Filestash

> Filestash web file manager with Collabora Online for in-browser office editing, using Docker Compose.
{.is-info}

_Last updated: Oct 2026_

## Overview

Filestash is a web file manager that connects to many backends (local disk, SFTP, S3, SMB, WebDAV and more). This stack pairs it with Collabora Online (CODE) as a WOPI server so you can open and edit Word, Excel and PowerPoint files in the browser.

**Why you'd want it:** A single web front end for all your storage, with document editing built in.

| Item | Value |
|---|---|
| Stack folder | `/opt/stacks/filestash` |
| Category | Documents |
| Main image | `machines/filestash:latest` |
| Services | 2 |
| Web UI | https://files.<YOUR_DOMAIN> |
| Official docs | https://www.filestash.app/docs/ |

## Requirements

- A Linux host with Docker Engine and the Docker Compose plugin (see [Docker](/docker))
- The shared folder layout and `.env`/`.secret` files from [Docker Folders](/dockerfolders) and [Environment](/environment)
- A reverse proxy with TLS for both Filestash and the Collabora endpoint
- At least 2 CPU cores and 2 GB RAM for Collabora

## Compose

Folder layout:

```text
/opt/stacks/
├── _shared/
│   ├── .env        # shared, non-secret settings
│   └── .secret     # passwords, tokens and keys (chmod 600, never commit)
└── filestash/
    └── compose.yaml
/opt/appdata/filestash/   # persistent application data
```

`/opt/stacks/filestash/compose.yaml`:

```yaml
services:
  filestash:
    container_name: filestash
    image: machines/filestash:latest
    env_file:
    - /opt/stacks/_shared/.env
    - /opt/stacks/_shared/.secret
    restart: always
    user: 0:0
    environment:
    - APPLICATION_URL=files.<YOUR_DOMAIN> # bare hostname only - no scheme, Filestash derives it from X-Forwarded-Proto
    - CANARY=true
    - OFFICE_URL=http://wopi_server:9980
    - OFFICE_FILESTASH_URL=http://filestash:8334
    - OFFICE_REWRITE_URL=https://files.<YOUR_DOMAIN>
    logging:
      driver: json-file
      options:
        max-file: ${DOCKERLOGGING_MAXFILE:-10}
        max-size: ${DOCKERLOGGING_MAXSIZE:-200k}
    ports:
    - 8334:8334
    volumes:
    - /opt/appdata/filestash/state:/app/data/state # Persistent storage for Filestash data

  wopi_server:
    container_name: filestash_wopi
    image: collabora/code:24.04.10.2.1
    env_file:
    - /opt/stacks/_shared/.env
    - /opt/stacks/_shared/.secret
    restart: always
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 2000M
    environment:
    - "extra_params=--o:ssl.enable=false"
    - "aliasgroup1=https://files.<YOUR_DOMAIN>:443"
    command:
    - /bin/bash
    - -c
    - |
        curl -o /usr/share/coolwsd/browser/dist/branding-desktop.css https://gist.githubusercontent.com/mickael-kerjean/bc1f57cd312cf04731d30185cc4e7ba2/raw/d706dcdf23c21441e5af289d871b33defc2770ea/destop.css
        /bin/su -s /bin/bash -c '/start-collabora-online.sh' cool
    user: root
    ports:
    - 9980:9980
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
| 8334 | 8334 | tcp | `filestash` | Filestash web UI |
| 9980 | 9980 | tcp | `wopi_server` | Collabora Online (WOPI) |

## Volumes

| Host path | Container path | Service | Purpose |
|---|---|---|---|
| `/opt/appdata/filestash/state` | `/app/data/state` | `filestash` | Persistent storage for Filestash data |

## Resource limits

Recommended limits, already set in the compose file under `deploy.resources`. Raise them if the app is starved, or lower them on small hosts.

| Service | CPU limit | Memory limit | Memory reservation | GPU |
|---|---|---|---|---|
| `wopi_server` | 2.0 | 2000M | - | - |

## Setup

1. Create the folders:
   ```bash
   sudo mkdir -p /opt/stacks/filestash /opt/appdata/filestash
   sudo chown -R $USER: /opt/stacks/filestash
   ```
2. Add the variables above to `/opt/stacks/_shared/.env` and `/opt/stacks/_shared/.secret` (`chmod 600 .secret`).
3. Save the compose file as `/opt/stacks/filestash/compose.yaml`.
4. Check the configuration, then start the stack:
   ```bash
   cd /opt/stacks/filestash
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret config --quiet
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
   ```
5. Watch the first start: `docker compose logs -f`.
6. Proxy `files.<YOUR_DOMAIN>` to port 8334 and pass WebSocket upgrades through.
7. Open the site and set the admin password at `/admin`.
8. In the admin console, enable the storage backends you want.
9. Under **Settings > Office**, confirm the Collabora URL, then open a `.docx` to test.

> Tip: create a small wrapper such as `alias dc='docker compose --env-file /opt/stacks/_shared/.env --env-file /opt/stacks/_shared/.secret'` so every stack uses the same env files.
{.is-success}

## Updating

```bash
cd /opt/stacks/filestash
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret pull
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
docker image prune -f
```

Some images here are pinned to a version. Read the upstream release notes before changing the tag.

Back up `/opt/appdata/filestash` before major upgrades. [Watchtower](/watchtower) can automate updates for stateless apps.

## Troubleshooting

- Check status and logs: `docker compose ps` and `docker compose logs --tail=100`.
- **Office documents open blank:** `aliasgroup1` must match the public HTTPS URL exactly, including `:443`.
- **Wrong links behind the proxy:** `APPLICATION_URL` is the bare host name only (no `https://`).
- **Collabora restarts repeatedly:** give it more memory or raise the resource limit.
- **A variable is empty or reported missing (`:?err`):** run the `config` command above to see which variable is not set in your env files.

## Related

- [Docker Stack Catalog](/docker-stacks)
- [File Browser](/filebrowser)
- [Double Commander](/doublecommander)
- [Directory Lister](/directorylister)

---
*Educational example. Review every path, port and credential before you deploy, and never publish your `.secret` file.*
