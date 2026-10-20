---
title: Immich
description: Immich self-hosted photo and video library (server, machine learning, Valkey, Postgres) with Docker Compose.
published: true
date: 2026-10-18T03:28:05.484Z
tags: docker, docker-compose, homelab, photos, postgres, backup, google-photos-alternative
editor: markdown
dateCreated: 2026-09-08T21:11:08.018Z
---

# Immich

> Immich self-hosted photo and video library (server, machine learning, Valkey, Postgres) with Docker Compose.
{.is-info}

_Last updated: Oct 2026_

## Overview

Immich is a self-hosted alternative to Google Photos. Phones back up automatically, and you get face recognition, smart search, maps, albums and sharing. The stack has four services: the server, machine learning, Valkey (Redis) and a Postgres build with vector extensions.

**Why you'd want it:** Keep your family photos at home with a modern app experience.

| Item | Value |
|---|---|
| Stack folder | `/opt/stacks/immich` |
| Category | Photos and cameras |
| Main image | `ghcr.io/immich-app/immich-server:release` |
| Services | 4 |
| Web UI | http://<SERVER_IP>:32283 |
| Official docs | https://immich.app/docs/install/docker-compose |

## Requirements

- A Linux host with Docker Engine and the Docker Compose plugin (see [Docker](/docker))
- The shared folder layout and `.env`/`.secret` files from [Docker Folders](/dockerfolders) and [Environment](/environment)
- SSD storage for the Postgres data folder
- Enough RAM for machine learning (3 GB limit recommended)
- Optional: OIDC provider such as [Keycloak](/keycloak) for single sign-on

## Compose

Folder layout:

```text
/opt/stacks/
├── _shared/
│   ├── .env        # shared, non-secret settings
│   └── .secret     # passwords, tokens and keys (chmod 600, never commit)
└── immich/
    └── compose.yaml
/opt/appdata/immich/   # persistent application data
```

`/opt/stacks/immich/compose.yaml`:

```yaml
#
# WARNING: To install Immich, follow our guide: https://immich.app/docs/install/docker-compose
#
# Make sure to use the docker-compose.yml of the current release:
#
# https://github.com/immich-app/immich/releases/latest/download/docker-compose.yml
#
# The compose file on main may not be compatible with the latest release.

name: immich

services:
  immich-server:
    image: ghcr.io/immich-app/immich-server:release
    container_name: immich_server
    # Add the user line here using your host UID:GID
    user: "${PUID}:${PGID}"
    env_file:
      - /opt/stacks/_shared/.env
      - /opt/stacks/_shared/.secret
    environment:
      # OAuth/OIDC with remote Keycloak (for LDAP federation):
      # 1. Deploy Immich and access the admin UI at http://your-domain:2283 (create an initial admin user via local auth).
      # 2. In Keycloak: Create a new OIDC client (confidential, authorization code grant) with redirect URIs like:
      #    - http://your-domain:2283/auth/login
      #    - http://your-domain:2283/user-settings
      #    - app.immich:///oauth-callback (for mobile apps)
      # 3. In Immich Admin UI (Administration > Settings > OAuth):
      #    - Enable: true
      #    - Issuer URL: https://your-keycloak-domain/realms/your-realm (append /.well-known/openid-configuration if needed)
      #    - Client ID: your-client-id
      #    - Client Secret: your-client-secret
      #    - Scope: openid email profile
      #    - Button Text: Login with Keycloak
      #    - Auto Register: true (to auto-create users from LDAP-synced Keycloak users)
      #    - Claims: Map 'preferred_username' to Storage Label, 'immich_role' to Role (user/admin), etc., for user attributes from LDAP.
      # 4. Ensure Keycloak's User Federation is configured for your remote LDAP (e.g., bind DN, users DN, sync users/groups).
      # See Immich docs: https://immich.app/docs/administration/oauth/
      # Keycloak LDAP guide: https://www.keycloak.org/docs/latest/server_admin/#_user_federation
    # extends:
    #   file: hwaccel.transcoding.yml
    #   service: cpu # set to one of [nvenc, quicksync, rkmpp, vaapi, vaapi-wsl] for accelerated transcoding
      - NODE_EXTRA_CA_CERTS=/certs/custom-ca.pem
    logging:
      driver: json-file
      options:
        max-file: ${DOCKERLOGGING_MAXFILE:-10}  # Max number of log files
        max-size: ${DOCKERLOGGING_MAXSIZE:-200k}  # Max file size
    volumes:
      # Do not edit the next line. If you want to change the media storage location on your system, edit the value of UPLOAD_LOCATION in the .env file
      #- /srv/photos/upload:/usr/src/app/upload
      - /srv/photos/photos:/data
      - /etc/localtime:/etc/localtime:ro
      #- /etc/pki/ca-trust/source/anchors/:/certs:ro
      - /opt/appdata/keycloak/certs/<YOUR_ROOT_CA>.pem:/certs/custom-ca.pem:ro
    ports:
      - 32283:2283

    depends_on:
      - redis
      - database
    restart: always
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 2000M
        reservations:
          memory: 500M
    healthcheck:
      disable: false

  immich-machine-learning:
    image: ghcr.io/immich-app/immich-machine-learning:release
    container_name: immich_machine_learning
    # For hardware acceleration, add one of -[armnn, cuda, rocm, openvino, rknn] to the image tag.
    # Example tag: ${IMMICH_VERSION:-release:?err}-cuda
    # extends: # uncomment this section for hardware acceleration - see https://immich.app/docs/features/ml-hardware-acceleration
    #   file: hwaccel.ml.yml
    #   service: cpu # set to one of [armnn, cuda, rocm, openvino, openvino-wsl, rknn] for accelerated inference - use the `-wsl` version for WSL2 where applicable
    volumes:
      - model-cache:/cache
    env_file:
      - /opt/stacks/_shared/.env
      - /opt/stacks/_shared/.secret
    restart: always
    deploy:
      resources:
        limits:
          cpus: '3.0'
          memory: 3000M
        reservations:
          memory: 1000M
    healthcheck:
      disable: false

  redis:
    image: docker.io/valkey/valkey:8-bookworm@sha256:fea8b3e67b15729d4bb70589eb03367bab9ad1ee89c876f54327fc7c6e618571
    container_name: immich_redis
    env_file:
      - /opt/stacks/_shared/.env
      - /opt/stacks/_shared/.secret
    healthcheck:
      test: redis-cli ping || exit 1
    restart: always
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 250M

  database:
    image: ghcr.io/immich-app/postgres:14-vectorchord0.4.3-pgvectors0.2.0@sha256:41eacbe83eca995561fe43814fd4891e16e39632806253848efaf04d3c8a8b84
    container_name: immich_postgres
    env_file:
      - /opt/stacks/_shared/.env
      - /opt/stacks/_shared/.secret
    environment:
      POSTGRES_DB: ${IM_POSTGRES_DB}
      POSTGRES_USER: ${IM_DB_USERNAME}
      POSTGRES_PASSWORD: ${IM_DB_PASSWORD}
      POSTGRES_INITDB_ARGS: '--data-checksums'
      # Uncomment the DB_STORAGE_TYPE: 'HDD' var if your database isn't stored on SSDs
      # DB_STORAGE_TYPE: 'HDD'
    volumes:
      # Do not edit the next line. If you want to change the database storage location on your system, edit the value of DB_DATA_LOCATION in the .env file
      - /opt/appdata/immich/postgres:/var/lib/postgresql/data
    shm_size: 128mb
    restart: always
    deploy:
      resources:
        limits:
          cpus: '1.5'
          memory: 1500M
        reservations:
          memory: 500M

volumes:
  model-cache:
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
IMMICH_VERSION=release
IM_POSTGRES_DB=immich
DB_HOSTNAME=database
DB_DATABASE_NAME=immich
```

`/opt/stacks/_shared/.secret`:

```ini
IM_DB_USERNAME=postgres
IM_DB_PASSWORD=<DB_PASSWORD>
DB_USERNAME=postgres
DB_PASSWORD=<DB_PASSWORD>
```

| Variable | File | Purpose | Example |
|---|---|---|---|
| `PUID` | `.env` | User ID the container runs as (see Docker Folders) | `1000` |
| `PGID` | `.env` | Group ID the container runs as | `1000` |
| `TZ` | `.env` | Time zone in tz database format, e.g. America/New_York | `<TZ>` |
| `DOCKERLOGGING_MAXFILE` | `.env` | Number of rotated container log files to keep | `10` |
| `DOCKERLOGGING_MAXSIZE` | `.env` | Maximum size of each container log file | `200k` |
| `IMMICH_VERSION` | `.env` | Immich release tag | `release` |
| `IM_POSTGRES_DB` | `.env` | Immich database name | `immich` |
| `IM_DB_USERNAME` | `.secret` | Immich database user | `postgres` |
| `IM_DB_PASSWORD` | `.secret` | Immich database password | `<DB_PASSWORD>` |
| `DB_HOSTNAME` | `.env` | Immich: database host (service name) | `database` |
| `DB_USERNAME` | `.secret` | Immich: database user | `postgres` |
| `DB_PASSWORD` | `.secret` | Immich: database password (same as IM_DB_PASSWORD) | `<DB_PASSWORD>` |
| `DB_DATABASE_NAME` | `.env` | Immich: database name | `immich` |

## Ports

| Host port | Container port | Protocol | Service | Purpose |
|---|---|---|---|---|
| 32283 | 2283 | tcp | `immich-server` | Web UI |

## Volumes

| Host path | Container path | Service | Purpose |
|---|---|---|---|
| `/srv/photos/photos` | `/data` | `immich-server` | Application data |
| `/etc/localtime` | `/etc/localtime` | `immich-server` | Host time zone (read-only) |
| `/opt/appdata/keycloak/certs/<YOUR_ROOT_CA>.pem` | `/certs/custom-ca.pem` | `immich-server` | Persistent data (read-only) |
| `model-cache (named volume)` | `/cache` | `immich-machine-learning` | Persistent data |
| `/opt/appdata/immich/postgres` | `/var/lib/postgresql/data` | `database` | Application data |

## Resource limits

Recommended limits, already set in the compose file under `deploy.resources`. Raise them if the app is starved, or lower them on small hosts.

| Service | CPU limit | Memory limit | Memory reservation | GPU |
|---|---|---|---|---|
| `immich-server` | 2.0 | 2000M | 500M | - |
| `immich-machine-learning` | 3.0 | 3000M | 1000M | - |
| `redis` | 0.5 | 250M | - | - |
| `database` | 1.5 | 1500M | 500M | - |

## Setup

1. Create the folders:
   ```bash
   sudo mkdir -p /opt/stacks/immich /opt/appdata/immich
   sudo chown -R $USER: /opt/stacks/immich
   ```
2. Add the variables above to `/opt/stacks/_shared/.env` and `/opt/stacks/_shared/.secret` (`chmod 600 .secret`).
3. Save the compose file as `/opt/stacks/immich/compose.yaml`.
4. Check the configuration, then start the stack:
   ```bash
   cd /opt/stacks/immich
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret config --quiet
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
   ```
5. Watch the first start: `docker compose logs -f`.
6. Set the database variables in `.env`/`.secret`. `DB_PASSWORD` and `IM_DB_PASSWORD` must match.
7. Start the stack and open `http://<SERVER_IP>:32283`. Create the admin user.
8. Install the mobile app, point it at your server URL and enable backup.
9. Optional OIDC: in **Administration > Settings > OAuth**, set the issuer URL, client ID and secret from Keycloak.

> Tip: create a small wrapper such as `alias dc='docker compose --env-file /opt/stacks/_shared/.env --env-file /opt/stacks/_shared/.secret'` so every stack uses the same env files.
{.is-success}

## Updating

```bash
cd /opt/stacks/immich
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret pull
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
docker image prune -f
```

Some images here are pinned to a version. Read the upstream release notes before changing the tag.

Back up `/opt/appdata/immich` before major upgrades. [Watchtower](/watchtower) can automate updates for stateless apps.

## Troubleshooting

- Check status and logs: `docker compose ps` and `docker compose logs --tail=100`.
- **Upgrade breaks the database:** never let Watchtower update `immich_postgres` or `immich_redis`. Follow the Immich release notes.
- **Custom CA for OIDC:** mount your root CA and set `NODE_EXTRA_CA_CERTS`, as shown.
- **Slow face recognition:** use a hardware-accelerated ML image tag (`-cuda`, `-openvino`).
- **A variable is empty or reported missing (`:?err`):** run the `config` command above to see which variable is not set in your env files.

## Related

- [Docker Stack Catalog](/docker-stacks)
- [Keycloak](/keycloak)
- [Watchtower](/watchtower)

---
*Educational example. Review every path, port and credential before you deploy, and never publish your `.secret` file.*
