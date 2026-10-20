---
title: Keycloak
description: Keycloak 26 identity provider and SSO (OIDC/SAML) behind a reverse proxy with PostgreSQL, using Docker Compose.
published: true
date: 2026-10-18T03:28:13.376Z
tags: docker, docker-compose, homelab, identity, ldap, postgres, oidc, sso
editor: markdown
dateCreated: 2026-09-08T21:26:12.838Z
---

# Keycloak

> Keycloak 26 identity provider and SSO (OIDC/SAML) behind a reverse proxy with PostgreSQL, using Docker Compose.
{.is-info}

_Last updated: Oct 2026_

## Overview

Keycloak is an identity and access management server. It provides single sign-on with OIDC and SAML, can federate users from LDAP or Active Directory, and supports MFA. Apps like Vaultwarden, Immich, Grafana and Proxmox can all log in through it.

**Why you'd want it:** One account and one MFA prompt for every app in your homelab.

| Item | Value |
|---|---|
| Stack folder | `/opt/stacks/keycloak` |
| Category | Identity and security |
| Main image | `quay.io/keycloak/keycloak:26.4.7` |
| Services | 1 |
| Web UI | https://keycloak.<YOUR_DOMAIN> |
| Official docs | https://www.keycloak.org/server/containers |

## Requirements

- A Linux host with Docker Engine and the Docker Compose plugin (see [Docker](/docker))
- The shared folder layout and `.env`/`.secret` files from [Docker Folders](/dockerfolders) and [Environment](/environment)
- PostgreSQL with a `keycloak` database
- A reverse proxy with TLS on the same host (`KC_PROXY_TRUSTED_ADDRESSES`)
- Optional: an LDAPS truststore in `/opt/appdata/keycloak/certs`

## Compose

Folder layout:

```text
/opt/stacks/
├── _shared/
│   ├── .env        # shared, non-secret settings
│   └── .secret     # passwords, tokens and keys (chmod 600, never commit)
└── keycloak/
    └── compose.yaml
/opt/appdata/keycloak/   # persistent application data
```

`/opt/stacks/keycloak/compose.yaml`:

```yaml
---
services:
  keycloak:
    image: quay.io/keycloak/keycloak:26.4.7 # Keycloak image
    container_name: keycloak
    hostname: keycloak
    env_file:
      - /opt/stacks/_shared/.env
      - /opt/stacks/_shared/.secret
    environment:
      # ───────────────────────────────────────────────────────────────
      #  1) Database — KC_DB_USERNAME/KC_DB_PASSWORD come from
      #     _shared/.env + .secret via env_file (names already match)
      # ───────────────────────────────────────────────────────────────
      KC_DB: postgres
      KC_DB_URL: jdbc:postgresql://<DB_HOST>:5432/keycloak

      # ───────────────────────────────────────────────────────────────
      #  2) Admin user — KC_BOOTSTRAP_ADMIN_USERNAME/PASSWORD come from
      #     _shared/.env + .secret via env_file (only used on first start)
      # ───────────────────────────────────────────────────────────────
      KC_HTTP_MANAGEMENT_PORT: 9001

      # ───────────────────────────────────────────────────────────────
      #  3) Hostname / proxy settings — KC_HOSTNAME comes from
      #     _shared/.env via env_file (very important behind reverse proxy!)
      # ───────────────────────────────────────────────────────────────
      KC_HOSTNAME_STRICT: true
      KC_HTTP_ENABLED: true           # TLS terminates at your reverse proxy
      KC_PROXY_HEADERS: xforwarded
      KC_PROXY_TRUSTED_ADDRESSES: 127.0.0.1/32   # reverse proxy on the same host

      # ───────────────────────────────────────────────────────────────
      #  4) LDAPS / Active Directory truststore  (your .p12 file)
      # ───────────────────────────────────────────────────────────────
      KC_TRUSTSTORE_PATHS: /opt/keycloak/conf/truststores
      # KC_TRUSTSTORE_PASSWORD comes from _shared/.secret via env_file (not interpolated here)
      # Optional – relaxes hostname check (only if SAN mismatch)
      KC_TLS_HOSTNAME_VERIFIER: DEFAULT #ANY/DEFAULT
      # If using Keycloak 20+ Quarkus distribution:
      #KC_SPI_TRUSTSTORE_FILE_FILE: /opt/keycloak/conf/truststores/ca_root.crt
      #KC_SPI_TRUSTSTORE_FILE_PASSWORD: <PASSWORD>
      #KC_SPI_TRUSTSTORE_FILE_HOSTNAME_VERIFICATION_POLICY: WILDCARD

      # ───────────────────────────────────────────────────────────────
      #  5) Logging / health / features
      # ───────────────────────────────────────────────────────────────
      KC_LOG_LEVEL: info
      KC_HEALTH_ENABLED: true
      KC_METRICS_ENABLED: true
      # KC_FEATURES:        token-exchange,organization   # add if needed
    logging:
      driver: json-file
      options:
        max-file: ${DOCKERLOGGING_MAXFILE:-10}  # Max number of log files
        max-size: ${DOCKERLOGGING_MAXSIZE:-200k}  # Max file size
    ports:
      - 38080:8080
    volumes:
      - /opt/appdata/keycloak/certs:/opt/keycloak/conf/truststores:ro
    command:
      - start
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 1500M

# Set default directory permissions (rwxr-xr-x)
#sudo find /opt/appdata/keycloak -type d -exec chmod 755 {} +
# Set default file permissions (rw-r--r--)
#sudo find /opt/appdata/keycloak -type f -exec chmod 644 {} +
# Tighten sensitive certs/secrets (rw-------)
#sudo chmod 644 /opt/appdata/keycloak/certs/*
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
KC_HOSTNAME=https://keycloak.<YOUR_DOMAIN>
KC_BOOTSTRAP_ADMIN_USERNAME=admin
```

`/opt/stacks/_shared/.secret`:

```ini
KC_DB_USERNAME=<DB_USER>
KC_DB_PASSWORD=<DB_PASSWORD>
KC_BOOTSTRAP_ADMIN_PASSWORD=<STRONG_PASSWORD>
KC_TRUSTSTORE_PASSWORD=<TRUSTSTORE_PASSWORD>
```

| Variable | File | Purpose | Example |
|---|---|---|---|
| `PUID` | `.env` | User ID the container runs as (see Docker Folders) | `1000` |
| `PGID` | `.env` | Group ID the container runs as | `1000` |
| `TZ` | `.env` | Time zone in tz database format, e.g. America/New_York | `<TZ>` |
| `DOCKERLOGGING_MAXFILE` | `.env` | Number of rotated container log files to keep | `10` |
| `DOCKERLOGGING_MAXSIZE` | `.env` | Maximum size of each container log file | `200k` |
| `KC_HOSTNAME` | `.env` | Public URL Keycloak is reached at | `https://keycloak.<YOUR_DOMAIN>` |
| `KC_DB_USERNAME` | `.secret` | Keycloak database user | `<DB_USER>` |
| `KC_DB_PASSWORD` | `.secret` | Keycloak database password | `<DB_PASSWORD>` |
| `KC_BOOTSTRAP_ADMIN_USERNAME` | `.env` | Initial Keycloak admin user (first start only) | `admin` |
| `KC_BOOTSTRAP_ADMIN_PASSWORD` | `.secret` | Initial Keycloak admin password (first start only) | `<STRONG_PASSWORD>` |
| `KC_TRUSTSTORE_PASSWORD` | `.secret` | Password for the LDAPS truststore (if used) | `<TRUSTSTORE_PASSWORD>` |

## Ports

| Host port | Container port | Protocol | Service | Purpose |
|---|---|---|---|---|
| 38080 | 8080 | tcp | `keycloak` | HTTP (put a TLS reverse proxy in front) |

## Volumes

| Host path | Container path | Service | Purpose |
|---|---|---|---|
| `/opt/appdata/keycloak/certs` | `/opt/keycloak/conf/truststores` | `keycloak` | Persistent data (read-only) |

## Resource limits

Recommended limits, already set in the compose file under `deploy.resources`. Raise them if the app is starved, or lower them on small hosts.

| Service | CPU limit | Memory limit | Memory reservation | GPU |
|---|---|---|---|---|
| `keycloak` | 2.0 | 1500M | - | - |

## Setup

1. Create the folders:
   ```bash
   sudo mkdir -p /opt/stacks/keycloak /opt/appdata/keycloak
   sudo chown -R $USER: /opt/stacks/keycloak
   ```
2. Add the variables above to `/opt/stacks/_shared/.env` and `/opt/stacks/_shared/.secret` (`chmod 600 .secret`).
3. Save the compose file as `/opt/stacks/keycloak/compose.yaml`.
4. Check the configuration, then start the stack:
   ```bash
   cd /opt/stacks/keycloak
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret config --quiet
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
   ```
5. Watch the first start: `docker compose logs -f`.
6. Set `KC_HOSTNAME`, `KC_DB_USERNAME`/`KC_DB_PASSWORD` and the bootstrap admin in `.env`/`.secret`.
7. Proxy `keycloak.<YOUR_DOMAIN>` to `http://127.0.0.1:38080`, passing `X-Forwarded-*` headers.
8. Sign in to the admin console, create a realm for your lab, and stop using the `master` realm for apps.
9. Optional: add LDAP under **User federation**.
10. Create one OIDC client per app.

> Tip: create a small wrapper such as `alias dc='docker compose --env-file /opt/stacks/_shared/.env --env-file /opt/stacks/_shared/.secret'` so every stack uses the same env files.
{.is-success}

## Updating

```bash
cd /opt/stacks/keycloak
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret pull
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
docker image prune -f
```

Some images here are pinned to a version. Read the upstream release notes before changing the tag.

Back up `/opt/appdata/keycloak` before major upgrades. [Watchtower](/watchtower) can automate updates for stateless apps.

## Troubleshooting

- Check status and logs: `docker compose ps` and `docker compose logs --tail=100`.
- **"HTTPS required" or redirect loops:** the proxy must send `X-Forwarded-Proto` and `X-Forwarded-Host`, and `KC_HOSTNAME` must match the public URL.
- **LDAPS certificate errors:** put your CA in the truststore folder and set `KC_TRUSTSTORE_PASSWORD`.
- Health and metrics are on management port 9001: `curl http://localhost:9001/health/ready` inside the container.
- **A variable is empty or reported missing (`:?err`):** run the `config` command above to see which variable is not set in your env files.

## Related

- [Docker Stack Catalog](/docker-stacks)
- [Vaultwarden](/vaultwarden)
- [Immich](/immich)
- [Apache Guacamole](/guacamole)

---
*Educational example. Review every path, port and credential before you deploy, and never publish your `.secret` file.*
