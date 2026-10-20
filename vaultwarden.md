---
title: Vaultwarden
description: Vaultwarden Bitwarden-compatible password manager with Keycloak SSO and PostgreSQL, using Docker Compose.
published: true
date: 2026-10-18T03:28:28.916Z
tags: docker, docker-compose, homelab, identity, postgres, security, sso, bitwarden, passwords
editor: markdown
dateCreated: 2026-09-08T21:11:08.018Z
---

# Vaultwarden

> Vaultwarden Bitwarden-compatible password manager with Keycloak SSO and PostgreSQL, using Docker Compose.
{.is-info}

_Last updated: Oct 2026_

## Overview

Vaultwarden is a lightweight server that is compatible with the official Bitwarden apps and browser extensions. This example uses PostgreSQL, SMTP for invites, and SSO through Keycloak (OIDC) with password login disabled.

**Why you'd want it:** Your own password vault for the whole family, using the polished Bitwarden clients.

| Item | Value |
|---|---|
| Stack folder | `/opt/stacks/vaultwarden` |
| Category | Identity and security |
| Main image | `vaultwarden/server:latest` |
| Services | 1 |
| Web UI | https://vault.<YOUR_DOMAIN> |
| Official docs | https://github.com/dani-garcia/vaultwarden/wiki |

## Requirements

- A Linux host with Docker Engine and the Docker Compose plugin (see [Docker](/docker))
- The shared folder layout and `.env`/`.secret` files from [Docker Folders](/dockerfolders) and [Environment](/environment)
- HTTPS. Bitwarden clients refuse plain HTTP, so use a reverse proxy with a valid certificate
- PostgreSQL with a `vaultwarden` database (or remove `DATABASE_URL` for SQLite)
- For SSO: a Keycloak realm with a confidential `vaultwarden` client

## Compose

Folder layout:

```text
/opt/stacks/
├── _shared/
│   ├── .env        # shared, non-secret settings
│   └── .secret     # passwords, tokens and keys (chmod 600, never commit)
└── vaultwarden/
    └── compose.yaml
/opt/appdata/vaultwarden/   # persistent application data
```

`/opt/stacks/vaultwarden/compose.yaml`:

```yaml
services:
  vaultwarden:
    env_file:
      - /opt/stacks/_shared/.env
      - /opt/stacks/_shared/.secret
    image: vaultwarden/server:latest
    container_name: vaultwarden
    hostname: vaultwarden
    environment:
      # Domain (must be publicly accessible for OIDC callbacks)
      - DOMAIN=${VW_DOMAIN}

      # Email Setup
      - SMTP_HOST=${EM_HOST}
      - SMTP_FROM=${EM_FROM}
      - SMTP_PORT=${EM_PORT}
      - SMTP_SECURITY=off
      - SMTP_SSL=false
      # Use these only if your server requires authentication
      #- SMTP_USERNAME=your_username
      #- SMTP_PASSWORD=<PASSWORD>
      # Database Setup
      - DATABASE_URL=postgresql://<DB_USER>:<DB_PASSWORD>@<DB_HOST>:5432/vaultwarden?sslmode=prefer

      # General settings
      - SIGNUPS_ALLOWED=false          # Recommended with SSO

      # Admin page — ADMIN_TOKEN comes from _shared/.secret via env_file

      # SSO / OIDC with Keycloak
      - SSO_ENABLED=true
      - SSO_ONLY=true                # Disable email/password login (recommended)
      - SSO_AUTHORITY=https://keycloak.<YOUR_DOMAIN>/realms/<REALM> #/.well-known/openid-configuration  # No trailing slash, exact issuer match
      - SSO_CLIENT_ID=vaultwarden
      # SSO_CLIENT_SECRET comes from _shared/.secret via env_file
      - SSO_SCOPES=openid profile email offline_access
      - SSO_PKCE=true
      - SSO_AUTH_ONLY_NOT_SESSION=true
      - SSO_CLIENT_CACHE_EXPIRATION=3600     # 1 hour cache for discovery
       # Optional but recommended
      - SSO_SIGNUPS_MATCH_EMAIL=true
      #- SSO_ALLOW_UNKNOWN_EMAIL_VERIFICATION=false

      - WEBSOCKET_ENABLED=true  # If you need live sync (requires reverse proxy support)

    logging:
      driver: json-file
      options:
        max-file: ${DOCKERLOGGING_MAXFILE:-10}  # Max number of log files
        max-size: ${DOCKERLOGGING_MAXSIZE:-200k}  # Max file size
    volumes:
      - /opt/appdata/vaultwarden/data:/data
    ports:
      - 30032:80
    restart: unless-stopped
```

## Environment variables

Compose reads these from the shared files with `--env-file`. Replace every `<PLACEHOLDER>` with your own value.

`/opt/stacks/_shared/.env`:

```ini
PUID=1000
PGID=1000
TZ=<TZ>
VW_DOMAIN=https://vault.<YOUR_DOMAIN>
EM_HOST=<SMTP_HOST>
EM_FROM=<SENDER_ADDRESS>
EM_PORT=587
DOCKERLOGGING_MAXFILE=10
DOCKERLOGGING_MAXSIZE=200k
```

`/opt/stacks/_shared/.secret`:

```ini
ADMIN_TOKEN=<ARGON2_HASH>
SSO_CLIENT_SECRET=<OIDC_CLIENT_SECRET>
```

| Variable | File | Purpose | Example |
|---|---|---|---|
| `PUID` | `.env` | User ID the container runs as (see Docker Folders) | `1000` |
| `PGID` | `.env` | Group ID the container runs as | `1000` |
| `TZ` | `.env` | Time zone in tz database format, e.g. America/New_York | `<TZ>` |
| `VW_DOMAIN` | `.env` | Public HTTPS URL of Vaultwarden | `https://vault.<YOUR_DOMAIN>` |
| `EM_HOST` | `.env` | SMTP relay host | `<SMTP_HOST>` |
| `EM_FROM` | `.env` | Sender address for notification mail | `<SENDER_ADDRESS>` |
| `EM_PORT` | `.env` | SMTP port | `587` |
| `DOCKERLOGGING_MAXFILE` | `.env` | Number of rotated container log files to keep | `10` |
| `DOCKERLOGGING_MAXSIZE` | `.env` | Maximum size of each container log file | `200k` |
| `ADMIN_TOKEN` | `.secret` | Vaultwarden /admin token (use an Argon2 hash) | `<ARGON2_HASH>` |
| `SSO_CLIENT_SECRET` | `.secret` | OIDC client secret from Keycloak | `<OIDC_CLIENT_SECRET>` |

## Ports

| Host port | Container port | Protocol | Service | Purpose |
|---|---|---|---|---|
| 30032 | 80 | tcp | `vaultwarden` | Web UI |

## Volumes

| Host path | Container path | Service | Purpose |
|---|---|---|---|
| `/opt/appdata/vaultwarden/data` | `/data` | `vaultwarden` | Application data |

## Resource limits

No limits are set. This app is light, so add `deploy.resources.limits` only if you need to cap it.

## Setup

1. Create the folders:
   ```bash
   sudo mkdir -p /opt/stacks/vaultwarden /opt/appdata/vaultwarden
   sudo chown -R $USER: /opt/stacks/vaultwarden
   ```
2. Add the variables above to `/opt/stacks/_shared/.env` and `/opt/stacks/_shared/.secret` (`chmod 600 .secret`).
3. Save the compose file as `/opt/stacks/vaultwarden/compose.yaml`.
4. Check the configuration, then start the stack:
   ```bash
   cd /opt/stacks/vaultwarden
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret config --quiet
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
   ```
5. Watch the first start: `docker compose logs -f`.
6. Generate an admin token hash: `docker run --rm -it vaultwarden/server /vaultwarden hash` and put it in `.secret` as `ADMIN_TOKEN`.
7. Proxy `vault.<YOUR_DOMAIN>` to port 30032.
8. In Keycloak, create the `vaultwarden` client with redirect URI `https://vault.<YOUR_DOMAIN>/identity/connect/oidc-signin`.
9. Open `/admin`, review the settings, and invite users. With `SSO_ONLY=true`, users sign in through Keycloak.

> Tip: create a small wrapper such as `alias dc='docker compose --env-file /opt/stacks/_shared/.env --env-file /opt/stacks/_shared/.secret'` so every stack uses the same env files.
{.is-success}

## Updating

```bash
cd /opt/stacks/vaultwarden
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret pull
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
docker image prune -f
```

Back up `/opt/appdata/vaultwarden` before major upgrades. [Watchtower](/watchtower) can automate updates for stateless apps.

## Troubleshooting

- Check status and logs: `docker compose ps` and `docker compose logs --tail=100`.
- **SSO "issuer mismatch":** `SSO_AUTHORITY` must equal the realm issuer exactly, with no trailing slash.
- **Invites not delivered:** check the SMTP settings in `/admin` and send a test email.
- **Back up `/opt/appdata/vaultwarden/data` and the database regularly.** This is your password vault.
- **A variable is empty or reported missing (`:?err`):** run the `config` command above to see which variable is not set in your env files.

## Related

- [Docker Stack Catalog](/docker-stacks)
- [Keycloak](/keycloak)

---
*Educational example. Review every path, port and credential before you deploy, and never publish your `.secret` file.*
