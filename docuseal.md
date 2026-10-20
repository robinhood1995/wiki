---
title: DocuSeal
description: Self-hosted document signing (DocuSign alternative) with DocuSeal and Docker Compose.
published: true
date: 2026-10-18T03:27:23.149Z
tags: docker, docker-compose, homelab, documents, e-signature, postgres
editor: markdown
dateCreated: 2026-09-08T21:15:35.866Z
---

# DocuSeal

> Self-hosted document signing (DocuSign alternative) with DocuSeal and Docker Compose.
{.is-info}

_Last updated: Oct 2026_

## Overview

DocuSeal is an open-source document signing platform. Upload a PDF, place signature and form fields, and send it out for signing. Completed documents are stored with an audit trail.

**Why you'd want it:** Sign leases, contracts and permission slips without paying per envelope or handing documents to a third party.

| Item | Value |
|---|---|
| Stack folder | `/opt/stacks/docuseal` |
| Category | Documents |
| Main image | `docuseal/docuseal:latest` |
| Services | 1 |
| Web UI | https://docuseal.<YOUR_DOMAIN> |
| Official docs | https://www.docuseal.com/docs |

## Requirements

- A Linux host with Docker Engine and the Docker Compose plugin (see [Docker](/docker))
- The shared folder layout and `.env`/`.secret` files from [Docker Folders](/dockerfolders) and [Environment](/environment)
- A PostgreSQL database (a dedicated `docuseal` database and user)
- An SMTP relay so DocuSeal can email signing links
- A reverse proxy with TLS (`FORCE_SSL=true` expects HTTPS)

## Compose

Folder layout:

```text
/opt/stacks/
├── _shared/
│   ├── .env        # shared, non-secret settings
│   └── .secret     # passwords, tokens and keys (chmod 600, never commit)
└── docuseal/
    └── compose.yaml
/opt/appdata/docuseal/   # persistent application data
```

`/opt/stacks/docuseal/compose.yaml`:

```yaml
﻿services:
  app:
    image: docuseal/docuseal:latest
    env_file:
      - /opt/stacks/_shared/.env
      - /opt/stacks/_shared/.secret
    ports:
      - 3000:3000
    volumes:
      - /opt/appdata/docuseal:/data/docuseal
    environment:
      - HOST=docuseal.<YOUR_DOMAIN>
      - FORCE_SSL=true
      - DATABASE_URL=postgresql://<DB_USER>:<DB_PASSWORD>@<DB_HOST>:5432/docuseal
      - SMTP_ADDRESS=<SMTP_HOST>
      - SMTP_PORT=25
      - SMTP_FROM=<ADMIN_EMAIL>
      - SMTP_ENABLE_STARTTLS=false
```

## Environment variables

Compose reads these from the shared files with `--env-file`. Replace every `<PLACEHOLDER>` with your own value.

`/opt/stacks/_shared/.env`:

```ini
PUID=1000
PGID=1000
TZ=<TZ>
```

| Variable | File | Purpose | Example |
|---|---|---|---|
| `PUID` | `.env` | User ID the container runs as (see Docker Folders) | `1000` |
| `PGID` | `.env` | Group ID the container runs as | `1000` |
| `TZ` | `.env` | Time zone in tz database format, e.g. America/New_York | `<TZ>` |

## Ports

| Host port | Container port | Protocol | Service | Purpose |
|---|---|---|---|---|
| 3000 | 3000 | tcp | `app` | Web UI |

## Volumes

| Host path | Container path | Service | Purpose |
|---|---|---|---|
| `/opt/appdata/docuseal` | `/data/docuseal` | `app` | Application data |

## Resource limits

No limits are set. This app is light, so add `deploy.resources.limits` only if you need to cap it.

## Setup

1. Create the folders:
   ```bash
   sudo mkdir -p /opt/stacks/docuseal /opt/appdata/docuseal
   sudo chown -R $USER: /opt/stacks/docuseal
   ```
2. Add the variables above to `/opt/stacks/_shared/.env` and `/opt/stacks/_shared/.secret` (`chmod 600 .secret`).
3. Save the compose file as `/opt/stacks/docuseal/compose.yaml`.
4. Check the configuration, then start the stack:
   ```bash
   cd /opt/stacks/docuseal
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret config --quiet
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
   ```
5. Watch the first start: `docker compose logs -f`.
6. Create the database: `CREATE DATABASE docuseal; CREATE USER docuseal WITH PASSWORD '<DB_PASSWORD>'; GRANT ALL ON DATABASE docuseal TO docuseal;`
7. Point your reverse proxy at `http://<SERVER_IP>:3000`.
8. Open the site and create the first admin account.
9. Under **Settings > Email**, confirm the SMTP settings and send a test.

> Tip: create a small wrapper such as `alias dc='docker compose --env-file /opt/stacks/_shared/.env --env-file /opt/stacks/_shared/.secret'` so every stack uses the same env files.
{.is-success}

## Updating

```bash
cd /opt/stacks/docuseal
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret pull
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
docker image prune -f
```

Back up `/opt/appdata/docuseal` before major upgrades. [Watchtower](/watchtower) can automate updates for stateless apps.

## Troubleshooting

- Check status and logs: `docker compose ps` and `docker compose logs --tail=100`.
- **Redirect loop:** `FORCE_SSL=true` needs the proxy to send `X-Forwarded-Proto: https`.
- **Emails not sent:** check `SMTP_ADDRESS`, `SMTP_PORT` and whether your relay needs STARTTLS or authentication.
- **Database errors on start:** verify `DATABASE_URL` and that the database user owns the `public` schema.
- **A variable is empty or reported missing (`:?err`):** run the `config` command above to see which variable is not set in your env files.

## Related

- [Docker Stack Catalog](/docker-stacks)
- [Keycloak](/keycloak)
- [Filestash](/filestash)
- [Vaultwarden](/vaultwarden)

---
*Educational example. Review every path, port and credential before you deploy, and never publish your `.secret` file.*
