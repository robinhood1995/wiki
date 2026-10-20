---
title: RustDesk Server
description: Self-hosted RustDesk server (hbbs ID/rendezvous + hbbr relay) for your own remote desktop network, with Docker Compose.
published: true
date: 2026-10-20T20:01:44.736Z
tags: docker, docker-compose, homelab, identity, remote-access, remote-desktop, rustdesk
editor: markdown
dateCreated: 2026-04-24T19:43:25.326Z
---

# RustDesk Server

> Self-hosted RustDesk server (hbbs ID/rendezvous + hbbr relay) for your own remote desktop network, with Docker Compose.
{.is-info}

_Last updated: Oct 2026_

## Overview

RustDesk is an open-source remote desktop app similar to TeamViewer or AnyDesk. By running your own server, the clients use your hbbs (ID and rendezvous) and hbbr (relay) instead of the public servers. Connections are end-to-end encrypted with a key pair generated on first start.

**Why you'd want it:** Help family members or reach your own PCs remotely without relying on, or paying for, a third-party service.

| Item | Value |
|---|---|
| Stack folder | `/opt/stacks/rustdesk` |
| Category | Identity and security |
| Main image | `rustdesk/rustdesk-server:latest` |
| Services | 2 |
| Web UI | None (configure the RustDesk clients) |
| Official docs | https://rustdesk.com/docs/en/self-host/rustdesk-server-oss/docker/ |

## Requirements

- A Linux host with Docker Engine and the Docker Compose plugin (see [Docker](/docker))
- The shared folder layout and `.env`/`.secret` files from [Docker Folders](/dockerfolders) and [Environment](/environment)
- A public IP or DNS name for the server
- Port forwards on your router: TCP 21115-21117 and UDP 21116 (21118-21119 only for the web client)

## Compose

Folder layout:

```text
/opt/stacks/
├── _shared/
│   ├── .env        # shared, non-secret settings
│   └── .secret     # passwords, tokens and keys (chmod 600, never commit)
└── rustdesk/
    └── compose.yaml
/opt/appdata/rustdesk/   # persistent application data
```

`/opt/stacks/rustdesk/compose.yaml`:

```yaml
services:
  hbbs: # ID / signaling server
    container_name: hbbs
    image: rustdesk/rustdesk-server:latest
    command: hbbs -r <RELAY_HOST>:21117   # public DNS name or IP of this server
    env_file:
      - /opt/stacks/_shared/.env
      - /opt/stacks/_shared/.secret
    logging:
      driver: json-file
      options:
        max-file: ${DOCKERLOGGING_MAXFILE:-10}  # Max number of log files
        max-size: ${DOCKERLOGGING_MAXSIZE:-200k}  # Max file size
    ports:
      - "21115:21115"
      - "21116:21116"
      - "21116:21116/udp"
      - "21118:21118"
    volumes:
      - /opt/appdata/rustdesk/data:/root
    depends_on:
      - hbbr
    restart: unless-stopped
  hbbr: # Relay server
    container_name: hbbr
    image: rustdesk/rustdesk-server:latest
    command: hbbr
    env_file:
      - /opt/stacks/_shared/.env
      - /opt/stacks/_shared/.secret
    logging:
      driver: json-file
      options:
        max-file: ${DOCKERLOGGING_MAXFILE:-10}  # Max number of log files
        max-size: ${DOCKERLOGGING_MAXSIZE:-200k}  # Max file size
    ports:
      - "21117:21117"
      - "21119:21119"
    volumes:
      - /opt/appdata/rustdesk/data:/root
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
| 21115 | 21115 | tcp | `hbbs` | NAT type test (hbbs) |
| 21116 | 21116 | tcp | `hbbs` | ID registration and hole punching (TCP + UDP) |
| 21116 | 21116 | udp | `hbbs` | ID registration and hole punching (TCP + UDP) |
| 21118 | 21118 | tcp | `hbbs` | Web client support (hbbs) |
| 21117 | 21117 | tcp | `hbbr` | Relay (hbbr) |
| 21119 | 21119 | tcp | `hbbr` | Web client support (hbbr) |

## Volumes

| Host path | Container path | Service | Purpose |
|---|---|---|---|
| `/opt/appdata/rustdesk/data` | `/root` | `hbbs` | Server keys (`id_ed25519`, `id_ed25519.pub`) and database |
| `/opt/appdata/rustdesk/data` | `/root` | `hbbr` | Server keys (`id_ed25519`, `id_ed25519.pub`) and database |

## Resource limits

No limits are set. This app is light, so add `deploy.resources.limits` only if you need to cap it.

## Setup

1. Create the folders:
   ```bash
   sudo mkdir -p /opt/stacks/rustdesk /opt/appdata/rustdesk
   sudo chown -R $USER: /opt/stacks/rustdesk
   ```
2. Add the variables above to `/opt/stacks/_shared/.env` and `/opt/stacks/_shared/.secret` (`chmod 600 .secret`).
3. Save the compose file as `/opt/stacks/rustdesk/compose.yaml`.
4. Replace `<RELAY_HOST>` in the `hbbs` command with the public DNS name or IP your clients will use.
5. Check the configuration, then start the stack:
   ```bash
   cd /opt/stacks/rustdesk
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret config --quiet
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
   ```
6. Watch the first start: `docker compose logs -f`.
7. On first start hbbs creates a key pair in `/opt/appdata/rustdesk/data`.
8. Copy the public key: `cat /opt/appdata/rustdesk/data/id_ed25519.pub`.
9. In each RustDesk client, open **Settings > Network > ID/Relay server** and enter `<RELAY_HOST>` as the ID server and relay server, plus the public key.

> Tip: create a small wrapper such as `alias dc='docker compose --env-file /opt/stacks/_shared/.env --env-file /opt/stacks/_shared/.secret'` so every stack uses the same env files.
{.is-success}

## Updating

```bash
cd /opt/stacks/rustdesk
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret pull
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
docker image prune -f
```

Back up `/opt/appdata/rustdesk` before major upgrades. [Watchtower](/watchtower) can automate updates for stateless apps.

## Troubleshooting

- Check status and logs: `docker compose ps` and `docker compose logs --tail=100`.
- **"Key mismatch":** the client key must match `id_ed25519.pub`; do not delete the data folder or every client must be updated.
- **Clients connect but stay on "relay":** UDP 21116 is not forwarded, so direct (hole-punched) connections fail.
- **Offline status in clients:** check that TCP 21116 is reachable from outside, and that `<RELAY_HOST>` resolves to your public IP.
- Never share `id_ed25519` (the private key).
- **A variable is empty or reported missing (`:?err`):** run the `config` command above to see which variable is not set in your env files.

## Related

- [Docker Stack Catalog](/docker-stacks)
- [Apache Guacamole](/guacamole)
- [Keycloak](/keycloak)

---
*Educational example. Review every path, port and credential before you deploy, and never publish your `.secret` file.*
