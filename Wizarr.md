---
title: Wizarr
description: Wizarr user invitation portal for Plex and Jellyfin: standalone compose or as part of the Arr Stack.
published: true
date: 2026-10-18T03:27:17.976Z
tags: docker, docker-compose, homelab, invitations, jellyfin, media, plex
editor: markdown
dateCreated: 2025-12-14T04:51:54.874Z
---

# Wizarr

> Wizarr user invitation portal for Plex and Jellyfin: standalone compose or as part of the Arr Stack.
{.is-info}

_Last updated: Oct 2026_

## Overview
Wizarr is an invitation and onboarding portal for Plex, Jellyfin and Emby. You create a link, share it, and new users sign up and get walked through installing an app and connecting to your server.

**Why you'd want it:** Onboard friends and family without creating accounts by hand.

> **Where to find the full guide:** Wizarr is documented as part of the **[Arr Stack](/arr-stack)**, where it runs on the shared Gluetun network next to Sonarr and Radarr. A standalone stack also exists. Use the compose file below if you want Wizarr on its own. Run only one copy, because both use port 5690.
{.is-warning}

| Item | Value |
|---|---|
| Stack folder | `/opt/stacks/wizarr` (standalone) or inside `/opt/stacks/arr_stack` |
| Category | Media |
| Main image | `ghcr.io/wizarrrr/wizarr:latest` |
| Web UI | http://<SERVER_IP>:5690 |
| Official docs | https://docs.wizarr.dev/ |

## Requirements
- A Linux host with Docker Engine and the Docker Compose plugin (see [Docker](/docker))
- A Plex, Jellyfin or Emby server

## Compose
Standalone `/opt/stacks/wizarr/compose.yaml`:

```yaml
services:
  wizarr:
    env_file:
    - /opt/stacks/_shared/.env
    - /opt/stacks/_shared/.secret
    container_name: wizarr
    restart: unless-stopped
    image: ghcr.io/wizarrrr/wizarr:latest
    ports:
      - 5690:5690
    environment:
      - APP_URL=https://wizarr.<YOUR_DOMAIN>
      - TZ=${TZ}
    volumes:
      - /opt/appdata/wizarr/database:/data/database
      - /opt/appdata/wizard/steps:/data/wizard_steps
```

## Environment variables
| Variable | File | Purpose | Example |
|---|---|---|---|
| `TZ` | `.env` | Time zone | `<TZ>` |
| `APP_URL` | compose | Public URL users open for invitations | `https://wizarr.<YOUR_DOMAIN>` |

## Ports
| Host port | Container port | Protocol | Service | Purpose |
|---|---|---|---|---|
| 5690 | 5690 | tcp | `wizarr` | Web UI |

## Volumes
| Host path | Container path | Service | Purpose |
|---|---|---|---|
| `/opt/appdata/wizarr/database` | `/data/database` | `wizarr` | Database |
| `/opt/appdata/wizard/steps` | `/data/wizard_steps` | `wizarr` | Custom onboarding steps |

## Resource limits
No limits are set. Wizarr is very light.

## Setup
1. `sudo mkdir -p /opt/stacks/wizarr /opt/appdata/wizarr/database /opt/appdata/wizard/steps`
2. Save the compose file, then run `docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d` in `/opt/stacks/wizarr`.
3. Open `http://<SERVER_IP>:5690` and create the admin account.
4. Under **Settings > Servers**, add your media server. Jellyfin needs an API key from **Dashboard > API Keys**.
5. Create an invitation and test it in a private browser window.

## Updating
```bash
cd /opt/stacks/wizarr
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret pull
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
```

## Troubleshooting
- **Port 5690 already in use:** you are probably running the Arr Stack copy too. Run only one.
- **Links point to the wrong address:** set `APP_URL` to the public URL users will open.

## Related
- [Docker Stack Catalog](/docker-stacks)
- [Arr Stack](/arr-stack)
- [Seerr](/seerr)
- [Plex](/plex)
- [Jellyfin](/jellyfin)

---
*Educational example. Review every path, port and credential before you deploy, and never publish your `.secret` file.*
