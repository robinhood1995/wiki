---
title: Docker Folders
description: Shared folder layout for the stack guides: /opt/stacks, /opt/appdata, /srv/media and /srv/data.
published: true
date: 2026-11-22T06:33:35.624Z
tags: 
editor: markdown
dateCreated: 2025-10-19T23:58:33.872Z
---

# Docker Folders

> Shared folder layout used by every stack guide on this wiki: `/opt/stacks`, `/opt/appdata`, `/srv/media` and `/srv/data`.
{.is-info}

_Last updated: Nov 2026_

## Folder structure

The compose guides assume the same paths on every machine, so a stack can move between hosts without rewriting bind mounts. Create these directories (or mount a NAS volume at the same paths):

```text
/opt/stacks/
├── _shared/
│   ├── .env        # shared, non-secret settings (see Environment)
│   └── .secret     # passwords, tokens and keys (chmod 600, never commit)
└── <name>/
    └── compose.yaml
/opt/appdata/<name>/    # persistent application data for that stack
/srv/media              # media libraries
/srv/data               # downloads and other bulk data
```

`<name>` is the stack folder, for example `plex` or `immich`.

These paths replace older Synology-style examples such as `/volume1/docker`. If you are on a NAS, mount or bind your volume so it appears at `/opt/stacks`, `/opt/appdata` and `/srv` — do not put `/volume1` into the compose files on this wiki.

## Users and permissions

Pick a user that will own the stack files and, where the image supports it, run the container processes. Put that user's IDs in `/opt/stacks/_shared/.env`:

```ini
PUID=1000
PGID=1000
TZ=<TZ>
```

`1000:1000` is a common default on desktop Linux. Use `id -u` and `id -g` on your host if your user is different, and keep `TZ=<TZ>` as a placeholder until you set a real time zone (for example `America/New_York`).

Example (run as a user who can sudo):

```bash
sudo mkdir -p /opt/stacks/_shared /opt/appdata /srv/media /srv/data
sudo chown -R "$USER:" /opt/stacks /opt/appdata
# Adjust /srv/media and /srv/data to the group that should write downloads and libraries.
install -m 600 /dev/null /opt/stacks/_shared/.env
install -m 600 /dev/null /opt/stacks/_shared/.secret
```

Then fill `.env` and `.secret` as described on [Environment](/environment).

## Starting a stack

```bash
cd /opt/stacks/<name>
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
```

Optional helper:

```bash
alias dc='docker compose --env-file /opt/stacks/_shared/.env --env-file /opt/stacks/_shared/.secret'
```

## Related

- [Environment](/environment)
- [Docker](/docker)
- [Docker Stack Catalog](/docker-stacks)
- [Portainer](/portainer)
- [Dockge](/dockge)

---
*Educational example. Review every path, port and credential before you deploy, and never publish your `.secret` file.*
