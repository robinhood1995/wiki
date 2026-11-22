---
title: UniFi Network Application
description: Self-hosted UniFi Network Application (controller) with MongoDB, using Docker Compose.
published: true
date: 2026-11-22T06:34:01.371Z
tags: docker, docker-compose, homelab, identity, network, unifi
editor: markdown
dateCreated: 2026-11-22T06:34:01.371Z
---

# UniFi Network Application

> Self-hosted UniFi Network Application (controller) with MongoDB, using Docker Compose.
{.is-info}

_Last updated: Nov 2026_

## Overview

The UniFi Network Application is the controller for UniFi gateways, switches and access points. Running it yourself keeps adoption, firmware and client history on your own host. This stack follows the LinuxServer image plus a dedicated MongoDB container.

**Why you'd want it:** Manage UniFi gear without depending on a cloud-only controller.

| Item | Value |
|---|---|
| Stack folder | `/opt/stacks/unifi` |
| Category | Identity and security |
| Main image | `lscr.io/linuxserver/unifi-network-application:latest` |
| Services | 2 |
| Web UI | https://<SERVER_IP>:8443 |
| Official docs | https://docs.linuxserver.io/images/docker-unifi-network-application/ |

## Requirements

- A Linux host with Docker Engine and the Docker Compose plugin (see [Docker](/docker))
- The shared folder layout and `.env`/`.secret` files from [Docker Folders](/dockerfolders) and [Environment](/environment)
- Enough memory for MongoDB plus the controller (2 GB RAM is a comfortable minimum)
- Devices must be able to reach the controller inform port (8080/tcp) on this host
- Check the LinuxServer docs for the MongoDB version their current tag supports; this example uses MongoDB 7

## Compose

Folder layout:

```text
/opt/stacks/
├── _shared/
│   ├── .env        # shared, non-secret settings
│   └── .secret     # passwords, tokens and keys (chmod 600, never commit)
└── unifi/
    └── compose.yaml
/opt/appdata/unifi/   # persistent application data
```

`/opt/stacks/unifi/compose.yaml`:

```yaml
---
services:
  unifi-db:
    image: docker.io/mongo:7.0
    container_name: unifi-db
    hostname: unifi-db
    env_file:
      - /opt/stacks/_shared/.env
      - /opt/stacks/_shared/.secret
    environment:
      - MONGO_INITDB_ROOT_USERNAME=${MONGO_ROOT_USER:?err}
      - MONGO_INITDB_ROOT_PASSWORD=${MONGO_ROOT_PASSWORD:?err}
      - MONGO_USER=${MONGO_USER:?err}
      - MONGO_PASS=${MONGO_PASS:?err}
      - MONGO_DBNAME=unifi
    logging:
      driver: json-file
      options:
        max-file: ${DOCKERLOGGING_MAXFILE:-10}
        max-size: ${DOCKERLOGGING_MAXSIZE:-200k}
    volumes:
      - /opt/appdata/unifi/db:/data/db
      - /opt/stacks/unifi/init-mongo.sh:/docker-entrypoint-initdb.d/init-mongo.sh:ro
    restart: unless-stopped

  unifi:
    image: lscr.io/linuxserver/unifi-network-application:latest
    container_name: unifi
    hostname: unifi
    depends_on:
      - unifi-db
    env_file:
      - /opt/stacks/_shared/.env
      - /opt/stacks/_shared/.secret
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
      - MONGO_USER=${MONGO_USER:?err}
      - MONGO_PASS=${MONGO_PASS:?err}
      - MONGO_HOST=unifi-db
      - MONGO_PORT=27017
      - MONGO_DBNAME=unifi
      - MONGO_AUTHSOURCE=admin
    logging:
      driver: json-file
      options:
        max-file: ${DOCKERLOGGING_MAXFILE:-10}
        max-size: ${DOCKERLOGGING_MAXSIZE:-200k}
    ports:
      - 8443:8443     # Web UI (HTTPS)
      - 8080:8080     # Device inform
      - 3478:3478/udp # STUN
      - 10001:10001/udp # Device discovery
    volumes:
      - /opt/appdata/unifi/config:/config
    restart: unless-stopped
```

`/opt/stacks/unifi/init-mongo.sh` (create this next to `compose.yaml`; the official LinuxServer image docs include a matching example):

```bash
#!/bin/bash
set -e
mongoadmin() {
  mongosh --username "$MONGO_INITDB_ROOT_USERNAME" --password "$MONGO_INITDB_ROOT_PASSWORD" --authenticationDatabase admin "$@"
}
mongoadmin --eval "db.getSiblingDB('admin').createUser({ user: '$MONGO_USER', pwd: '$MONGO_PASS', roles: [{ role: 'readWrite', db: '$MONGO_DBNAME' }, { role: 'readWrite', db: '${MONGO_DBNAME}_stat' }, { role: 'readWrite', db: '${MONGO_DBNAME}_audit' }] })"
```

If your MongoDB image still ships `mongo` instead of `mongosh`, change the helper in that script to match the image docs.

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

`/opt/stacks/_shared/.secret` (chmod 600):

```ini
MONGO_ROOT_USER=root
MONGO_ROOT_PASSWORD=<DB_ROOT_PASSWORD>
MONGO_USER=unifi
MONGO_PASS=<DB_PASSWORD>
```

| Variable | File | Purpose | Example |
|---|---|---|---|
| PUID | .env | User ID the container runs as (see Docker Folders) | 1000 |
| PGID | .env | Group ID the container runs as | 1000 |
| TZ | .env | Time zone in tz database format, e.g. America/New_York | <TZ> |
| DOCKERLOGGING_MAXFILE | .env | Number of rotated container log files to keep | 10 |
| DOCKERLOGGING_MAXSIZE | .env | Maximum size of each container log file | 200k |
| MONGO_ROOT_USER | .secret | MongoDB root user | root |
| MONGO_ROOT_PASSWORD | .secret | MongoDB root password | <DB_ROOT_PASSWORD> |
| MONGO_USER | .secret | Application database user | unifi |
| MONGO_PASS | .secret | Application database password | <DB_PASSWORD> |

## Ports

| Host port | Container port | Protocol | Service | Purpose |
|---|---|---|---|---|
| 8443 | 8443 | tcp | unifi | HTTPS web UI |
| 8080 | 8080 | tcp | unifi | Device inform |
| 3478 | 3478 | udp | unifi | STUN |
| 10001 | 10001 | udp | unifi | Layer-2 discovery |

## Volumes

| Host path | Container path | Service | Purpose |
|---|---|---|---|
| `/opt/appdata/unifi/config` | `/config` | unifi | Controller data |
| `/opt/appdata/unifi/db` | `/data/db` | unifi-db | MongoDB data |
| `/opt/stacks/unifi/init-mongo.sh` | `/docker-entrypoint-initdb.d/init-mongo.sh` | unifi-db | First-run user init |

## Resource limits

No limits are set. Give the host enough RAM for MongoDB; add `deploy.resources.limits` if you need to cap it.

## Setup

1. Create the folders:
   ```bash
   sudo mkdir -p /opt/stacks/unifi /opt/appdata/unifi
   sudo chown -R $USER: /opt/stacks/unifi
   ```
2. Add the variables above to `/opt/stacks/_shared/.env` and `/opt/stacks/_shared/.secret` (`chmod 600 .secret`).
3. Save the compose file as `/opt/stacks/unifi/compose.yaml`.
4. Check the configuration, then start the stack:
   ```bash
   cd /opt/stacks/unifi
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret config --quiet
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
   ```
5. Watch the first start: `docker compose logs -f`.

6. `chmod 755 /opt/stacks/unifi/init-mongo.sh`. The init script runs only on an empty database volume.
7. Open `https://<SERVER_IP>:8443` (the first certificate is self-signed) and complete the setup wizard. Adopt devices only after the inform URL points at this host.
8. Keep the UI off the public internet unless you put it behind a reverse proxy you trust. Device inform (8080/tcp) must stay reachable from your UniFi gear.

> Tip: create a small wrapper such as `alias dc='docker compose --env-file /opt/stacks/_shared/.env --env-file /opt/stacks/_shared/.secret'` so every stack uses the same env files.
{.is-success}
## Updating

```bash
cd /opt/stacks/unifi
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret pull
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
docker image prune -f
```

Back up `/opt/appdata/unifi` before major upgrades. [Watchtower](/watchtower) can automate updates for stateless apps.

## Troubleshooting

- Check status and logs: `docker compose ps` and `docker compose logs --tail=100`.
- **Controller cannot reach MongoDB:** `MONGO_HOST` must be the database service name (`unifi-db` in this example), and the passwords in `.secret` must match what the init script used on first start. Changing the password later does not re-run the init script.
- **Devices stay offline:** check that inform traffic reaches port 8080 and that the inform hostname or address is one the devices can resolve.
- **A variable is empty or reported missing (`:?err`):** run the `config` command above to see which variable is not set in your env files.

## Related

- [Docker Stack Catalog](/docker-stacks)
- [oauth2-proxy](/oauth2-proxy)
- [Prometheus](/prometheus)

---
*Educational example. Review every path, port and credential before you deploy, and never publish your `.secret` file.*
