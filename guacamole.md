---
title: Apache Guacamole
description: Apache Guacamole clientless RDP/SSH/VNC gateway with MySQL and LDAP, using Docker Compose.
published: true
date: 2026-10-18T03:28:02.807Z
tags: docker, docker-compose, homelab, identity, ldap, rdp, remote-access, ssh, vnc
editor: markdown
dateCreated: 2026-09-08T21:15:35.866Z
---

# Apache Guacamole

> Apache Guacamole clientless RDP/SSH/VNC gateway with MySQL and LDAP, using Docker Compose.
{.is-info}

_Last updated: Oct 2026_

## Overview

Guacamole is a clientless remote desktop gateway. It gives you RDP, SSH, VNC and Telnet sessions in a browser tab, with a central list of connections, session recording and optional LDAP login.

**Why you'd want it:** Reach every server and desktop in your lab from any browser, with one login.

| Item | Value |
|---|---|
| Stack folder | `/opt/stacks/guacamole` |
| Category | Identity and security |
| Main image | `guacamole/guacd:latest` |
| Services | 2 |
| Web UI | http://<SERVER_IP>:30006/guacamole |
| Official docs | https://guacamole.apache.org/doc/gug/guacamole-docker.html |

## Requirements

- A Linux host with Docker Engine and the Docker Compose plugin (see [Docker](/docker))
- The shared folder layout and `.env`/`.secret` files from [Docker Folders](/dockerfolders) and [Environment](/environment)
- MySQL/MariaDB (or PostgreSQL) for connection and user data
- Optional: an LDAP/Active Directory server
- A reverse proxy with TLS for remote use

## Compose

Folder layout:

```text
/opt/stacks/
├── _shared/
│   ├── .env        # shared, non-secret settings
│   └── .secret     # passwords, tokens and keys (chmod 600, never commit)
└── guacamole/
    └── compose.yaml
/opt/appdata/guacamole/   # persistent application data
```

`/opt/stacks/guacamole/compose.yaml`:

```yaml
---
services:
  guacd:
    image: guacamole/guacd:latest
    container_name: guacd
    env_file:
      - /opt/stacks/_shared/.env
      - /opt/stacks/_shared/.secret
    volumes:
    - /opt/appdata/guacamole/drive:/drive:rw
    - /opt/appdata/guacamole/record:/record:rw
    restart: unless-stopped

  guacamole:
    image: guacamole/guacamole:latest
    container_name: guacamole
    env_file:
      - /opt/stacks/_shared/.env
      - /opt/stacks/_shared/.secret
    depends_on:
    - guacd
    environment:
      # Guacamole connection to guacd
      - GUACD_HOSTNAME=guacd
      - GUACD_PORT=4822
      # PostgreSQL connection (remote)
      #- POSTGRESQL_DATABASE=guacamole
      #- POSTGRESQL_HOSTNAME=<DB_HOST>
      #- POSTGRESQL_PASSWORD=<PASSWORD>
      #- POSTGRESQL_USERNAME=<DB_USER>
      # Remote MySQL configuration (remote)
      #- MYSQL_HOSTNAME=${DB_MYSQL_HOST}
      - MYSQL_DATABASE=guacamole
      #- MYSQL_USERNAME=${DB_USER:?err}
      #- MYSQL_PASSWORD=${DB_PASS:?err}
      - RECORDING_SEARCH_PATH=/record

      # LDAP Authentication
      - LDAP_HOSTNAME=<DB_HOST>
      - LDAP_PORT=389 # or 636 for LDAPS
      - LDAP_ENCRYPTION_METHOD=none # or ssl, starttls
      - LDAP_USER_BASE_DN=ou=<ORG_UNIT>,dc=<DOMAIN>,dc=<TLD>
      - LDAP_USERNAME_ATTRIBUTE=uid # or sAMAccountName for Active Directory
      # LDAP_SEARCH_BIND_DN: cn=searchuser,dc=example,dc=com # Optional, if a dedicated search user is needed
      # LDAP_SEARCH_BIND_PASSWORD: <PASSWORD> # Optional
    logging:
      driver: json-file
      options:
        max-file: ${DOCKERLOGGING_MAXFILE:-10}  # Max number of log files
        max-size: ${DOCKERLOGGING_MAXSIZE:-200k}  # Max file size
    volumes:
      - /opt/appdata/guacamole/record:/record:rw
    ports:
## enable next line if not using nginx
##    - 8080:8080/tcp # Guacamole is on :8080/guacamole, not /.
## enable next line when using nginx
    - 30006:8080/tcp
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 1000M
```

## Environment variables

Compose reads these from the shared files with `--env-file`. Replace every `<PLACEHOLDER>` with your own value.

`/opt/stacks/_shared/.env`:

```ini
PUID=1000
PGID=1000
TZ=<TZ>
DB_MYSQL_HOST=<DB_HOST>
DOCKERLOGGING_MAXFILE=10
DOCKERLOGGING_MAXSIZE=200k
```

`/opt/stacks/_shared/.secret`:

```ini
DB_USER=<DB_USER>
DB_PASS=<DB_PASSWORD>
```

| Variable | File | Purpose | Example |
|---|---|---|---|
| `PUID` | `.env` | User ID the container runs as (see Docker Folders) | `1000` |
| `PGID` | `.env` | Group ID the container runs as | `1000` |
| `TZ` | `.env` | Time zone in tz database format, e.g. America/New_York | `<TZ>` |
| `DB_MYSQL_HOST` | `.env` | Hostname or IP of your MySQL/MariaDB server | `<DB_HOST>` |
| `DB_USER` | `.secret` | Database username | `<DB_USER>` |
| `DB_PASS` | `.secret` | Database password | `<DB_PASSWORD>` |
| `DOCKERLOGGING_MAXFILE` | `.env` | Number of rotated container log files to keep | `10` |
| `DOCKERLOGGING_MAXSIZE` | `.env` | Maximum size of each container log file | `200k` |

## Ports

| Host port | Container port | Protocol | Service | Purpose |
|---|---|---|---|---|
| 30006 | 8080 | tcp | `guacamole` | Web UI |

## Volumes

| Host path | Container path | Service | Purpose |
|---|---|---|---|
| `/opt/appdata/guacamole/drive` | `/drive` | `guacd` | Persistent data |
| `/opt/appdata/guacamole/record` | `/record` | `guacd` | Persistent data |
| `/opt/appdata/guacamole/record` | `/record` | `guacamole` | Persistent data |

## Resource limits

Recommended limits, already set in the compose file under `deploy.resources`. Raise them if the app is starved, or lower them on small hosts.

| Service | CPU limit | Memory limit | Memory reservation | GPU |
|---|---|---|---|---|
| `guacamole` | 1.0 | 1000M | - | - |

## Setup

1. Create the folders:
   ```bash
   sudo mkdir -p /opt/stacks/guacamole /opt/appdata/guacamole
   sudo chown -R $USER: /opt/stacks/guacamole
   ```
2. Add the variables above to `/opt/stacks/_shared/.env` and `/opt/stacks/_shared/.secret` (`chmod 600 .secret`).
3. Save the compose file as `/opt/stacks/guacamole/compose.yaml`.
4. Check the configuration, then start the stack:
   ```bash
   cd /opt/stacks/guacamole
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret config --quiet
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
   ```
5. Watch the first start: `docker compose logs -f`.
6. Initialise the database schema: `docker run --rm guacamole/guacamole /opt/guacamole/bin/initdb.sh --mysql > initdb.sql`, then import it into your database.
7. Uncomment and set the `MYSQL_*` lines (or switch to the PostgreSQL block).
8. Open `http://<SERVER_IP>:30006/guacamole` and sign in as `guacadmin` / `guacadmin`. Change that password right away.
9. Add connections under **Settings > Connections**.

> Tip: create a small wrapper such as `alias dc='docker compose --env-file /opt/stacks/_shared/.env --env-file /opt/stacks/_shared/.secret'` so every stack uses the same env files.
{.is-success}

## Updating

```bash
cd /opt/stacks/guacamole
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret pull
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
docker image prune -f
```

Back up `/opt/appdata/guacamole` before major upgrades. [Watchtower](/watchtower) can automate updates for stateless apps.

## Troubleshooting

- Check status and logs: `docker compose ps` and `docker compose logs --tail=100`.
- **404 at /:** Guacamole lives at `/guacamole` unless your proxy rewrites the path.
- **LDAP users can't log in:** check `LDAP_USER_BASE_DN` and use `sAMAccountName` for Active Directory.
- **Recordings missing:** `/record` must be writable and shared by guacd and guacamole.
- **A variable is empty or reported missing (`:?err`):** run the `config` command above to see which variable is not set in your env files.

## Related

- [Docker Stack Catalog](/docker-stacks)
- [Keycloak](/keycloak)
- [RustDesk Server](/Rustdesk)

---
*Educational example. Review every path, port and credential before you deploy, and never publish your `.secret` file.*
