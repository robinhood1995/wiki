---
title: Arr Download 1 (second qBittorrent instance)
description: A second, independent Gluetun + qBittorrent instance on separate ports with Docker Compose.
published: true
date: 2026-10-18T03:27:31.204Z
tags: docker, docker-compose, downloads, gluetun, homelab, qbittorrent, vpn
editor: markdown
dateCreated: 2026-10-18T03:26:52.550Z
---

# Arr Download 1 (second qBittorrent instance)

> A second, independent Gluetun + qBittorrent instance on separate ports with Docker Compose.
{.is-info}

_Last updated: Oct 2026_

## Overview

A second copy of the download stack with only qBittorrent, behind its own Gluetun tunnel on different ports (6882, 8091, 8889, 8389). Use it to split private trackers from public ones, or to spread load across two VPN connections.

**Why you'd want it:** Separate VPN servers, ports and settings for different kinds of torrents, without touching your main client.

| Item | Value |
|---|---|
| Stack folder | `/opt/stacks/arr_download_1` |
| Category | Downloads |
| Main image | `lscr.io/linuxserver/qbittorrent:latest` |
| Services | 2 |
| Web UI | http://<SERVER_IP>:8091 |
| Official docs | https://github.com/qdm12/gluetun-wiki |

## Requirements

- A Linux host with Docker Engine and the Docker Compose plugin (see [Docker](/docker))
- The shared folder layout and `.env`/`.secret` files from [Docker Folders](/dockerfolders) and [Environment](/environment)
- Everything from [Arr Download](/arr-download)
- A VPN plan that allows another simultaneous connection

## Compose

Folder layout:

```text
/opt/stacks/
├── _shared/
│   ├── .env        # shared, non-secret settings
│   └── .secret     # passwords, tokens and keys (chmod 600, never commit)
└── arr_download_1/
    └── compose.yaml
/opt/appdata/arr_download_1/   # persistent application data
```

`/opt/stacks/arr_download_1/compose.yaml`:

```yaml
networks:
  media-network:
    name: media-network
    driver: bridge
services:
  gluetun_1:
    image: qmcgaw/gluetun:v3
    container_name: gluetun_1
    cap_add:
      - NET_ADMIN
    devices:
      - /dev/net/tun:/dev/net/tun
    environment:
      - TZ=${TZ:?err}
      - UPDATER_PERIOD=24h
      - VPN_SERVICE_PROVIDER=protonvpn
      - VPN_TYPE=${VPN_TYPE:?err}
      - BLOCK_MALICIOUS=off
      - OPENVPN_USER=${OPENVPN_USERNAME:?err}+pmp
      - OPENVPN_PASSWORD=${OPENVPN_PASSWORD:?err}
      - OPENVPN_CIPHERS=AES-256-GCM
      - WIREGUARD_PRIVATE_KEY=${WIREGUARD_PRIVATE_KEY:?err}
      - PORT_FORWARD_ONLY=on
      - VPN_PORT_FORWARDING=on
      - VPN_PORT_FORWARDING_UP_COMMAND=/bin/sh -c 'wget -O- --retry-connrefused --post-data "json={\"listen_port\":{{PORTS}}}" http://127.0.0.1:8091/api/v2/app/setPreferences 2>&1'
      - SERVER_COUNTRIES=${SERVER_COUNTRIES:?err}
      - SHADOWSOCKS=on
      # Your VPN credentials and settings here (e.g., VPN_SERVICE_PROVIDER, VPN_TYPE)
      - FIREWALL_OUTBOUND_SUBNETS=<LAN_SUBNET> # <-- Your actual LAN subnet
      # Optional: Use Docker DNS for inter-container hostname resolution
    logging:
      driver: json-file
      options:
        max-file: ${DOCKERLOGGING_MAXFILE:?err}  # Max number of log files
        max-size: ${DOCKERLOGGING_MAXSIZE:?err}  # Max file size
    volumes:
      - ${DOCKERFOLDER:?err}/gluetun_1/config:/gluetun
    ports:
      - 6882:6882/tcp   # Torrent Port TCP: qBittorrent
      - 6882:6882/udp   # Torrent Port UDP: qBittorrent
      - 8091:8091/tcp   # WebUI Portal: qBittorrent
      - 8889:8889/tcp   # Local Network HTTP proxy: Gluetun
      - 8389:8389/tcp   # Local Network Shadowsocks: Gluetun
      - 8389:8389/udp   # Local Network Shadowsocks: Gluetun

    networks:
      - media-network
    restart: unless-stopped
    sysctls:
      - net.ipv6.conf.all.disable_ipv6=1

  qbittorrent_1:
    image: lscr.io/linuxserver/qbittorrent:latest
    container_name: qbittorrent_1
    depends_on:
      gluetun_1:
        condition: service_healthy
    environment:
      - PUID=${PUID:?err}    # User ID
      - PGID=${PGID:?err}    # Group ID
      - TZ=${TZ:?err}        # TimeZone
      - UMASK=${UMASK:?err}  # File Umask
      - WEBUI_PORT=8091
    logging:
      driver: json-file
      options:
        max-file: ${DOCKERLOGGING_MAXFILE:?err}  # Max number of log files
        max-size: ${DOCKERLOGGING_MAXSIZE:?err}  # Max file size
    volumes:
      - ${DOCKERFOLDER:?err}/qbittorrent_1/config:/config
      - ${TORRENTDOWNLOAD:?err}:/data/torrents
      - ${MEDIAFOLDER:?err}:/data/media
    network_mode: "service:gluetun_1"
    # Optional: Use extra_hosts for specific local services if needed
    # extra_hosts:
    # - "my-nas:<SERVER_IP>"
    # ... other app settings
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 2000M
```

## Environment variables

Compose reads these from the shared files with `--env-file`. Replace every `<PLACEHOLDER>` with your own value.

`/opt/stacks/_shared/.env`:

```ini
PUID=1000
PGID=1000
TZ=<TZ>
VPN_TYPE=wireguard
SERVER_COUNTRIES=<COUNTRY_1>,<COUNTRY_2>
DOCKERLOGGING_MAXFILE=10
DOCKERLOGGING_MAXSIZE=200k
DOCKERFOLDER=/opt/appdata
UMASK=002
TORRENTDOWNLOAD=/srv/data/torrents
MEDIAFOLDER=/srv/media
```

`/opt/stacks/_shared/.secret`:

```ini
OPENVPN_USERNAME=<VPN_USERNAME>
OPENVPN_PASSWORD=<VPN_PASSWORD>
WIREGUARD_PRIVATE_KEY=<WIREGUARD_PRIVATE_KEY>
```

| Variable | File | Purpose | Example |
|---|---|---|---|
| `PUID` | `.env` | User ID the container runs as (see Docker Folders) | `1000` |
| `PGID` | `.env` | Group ID the container runs as | `1000` |
| `TZ` | `.env` | Time zone in tz database format, e.g. America/New_York | `<TZ>` |
| `VPN_TYPE` | `.env` | Gluetun VPN protocol: openvpn or wireguard | `wireguard` |
| `OPENVPN_USERNAME` | `.secret` | VPN provider OpenVPN username | `<VPN_USERNAME>` |
| `OPENVPN_PASSWORD` | `.secret` | VPN provider OpenVPN password | `<VPN_PASSWORD>` |
| `WIREGUARD_PRIVATE_KEY` | `.secret` | VPN provider WireGuard private key | `<WIREGUARD_PRIVATE_KEY>` |
| `SERVER_COUNTRIES` | `.env` | Comma-separated VPN server countries | `<COUNTRY_1>,<COUNTRY_2>` |
| `DOCKERLOGGING_MAXFILE` | `.env` | Number of rotated container log files to keep | `10` |
| `DOCKERLOGGING_MAXSIZE` | `.env` | Maximum size of each container log file | `200k` |
| `DOCKERFOLDER` | `.env` | Base folder for persistent app data | `/opt/appdata` |
| `UMASK` | `.env` | File creation mask for new files | `002` |
| `TORRENTDOWNLOAD` | `.env` | Torrent download folder | `/srv/data/torrents` |
| `MEDIAFOLDER` | `.env` | Root of your media library (movies, tv, music, books) | `/srv/media` |

## Ports

| Host port | Container port | Protocol | Service | Purpose |
|---|---|---|---|---|
| 6882 | 6882 | tcp | `gluetun_1` | Torrent Port TCP: qBittorrent |
| 6882 | 6882 | udp | `gluetun_1` | Torrent Port UDP: qBittorrent |
| 8091 | 8091 | tcp | `gluetun_1` | WebUI Portal: qBittorrent |
| 8889 | 8889 | tcp | `gluetun_1` | Local Network HTTP proxy: Gluetun |
| 8389 | 8389 | tcp | `gluetun_1` | Local Network Shadowsocks: Gluetun |
| 8389 | 8389 | udp | `gluetun_1` | Local Network Shadowsocks: Gluetun |

> Apps with `network_mode: service:<gluetun>` share the VPN container's network, so their ports are published on the Gluetun service.
{.is-info}

## Volumes

| Host path | Container path | Service | Purpose |
|---|---|---|---|
| `${DOCKERFOLDER}/gluetun_1/config` | `/gluetun` | `gluetun_1` | Application configuration |
| `${DOCKERFOLDER}/qbittorrent_1/config` | `/config` | `qbittorrent_1` | Application configuration |
| `${TORRENTDOWNLOAD}` | `/data/torrents` | `qbittorrent_1` | Downloads |
| `${MEDIAFOLDER}` | `/data/media` | `qbittorrent_1` | Media library |

## Resource limits

Recommended limits, already set in the compose file under `deploy.resources`. Raise them if the app is starved, or lower them on small hosts.

| Service | CPU limit | Memory limit | Memory reservation | GPU |
|---|---|---|---|---|
| `qbittorrent_1` | 2.0 | 2000M | - | - |

## Setup

1. Create the folders:
   ```bash
   sudo mkdir -p /opt/stacks/arr_download_1 /opt/appdata/arr_download_1
   sudo chown -R $USER: /opt/stacks/arr_download_1
   ```
2. Add the variables above to `/opt/stacks/_shared/.env` and `/opt/stacks/_shared/.secret` (`chmod 600 .secret`).
3. Save the compose file as `/opt/stacks/arr_download_1/compose.yaml`.
4. Check the configuration, then start the stack:
   ```bash
   cd /opt/stacks/arr_download_1
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret config --quiet
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
   ```
5. Watch the first start: `docker compose logs -f`.
6. Start the stack and confirm `docker logs gluetun_1` shows a connected, healthy tunnel.
7. Open qBittorrent on port 8091 and change the temporary password from `docker logs qbittorrent_1`.
8. Add it to Sonarr/Radarr or [qui](/arr-stack) as a second download client.

> Tip: create a small wrapper such as `alias dc='docker compose --env-file /opt/stacks/_shared/.env --env-file /opt/stacks/_shared/.secret'` so every stack uses the same env files.
{.is-success}

## Updating

```bash
cd /opt/stacks/arr_download_1
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret pull
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
docker image prune -f
```

Some images here are pinned to a version. Read the upstream release notes before changing the tag.

Back up `/opt/appdata/arr_download_1` before major upgrades. [Watchtower](/watchtower) can automate updates for stateless apps.

## Troubleshooting

- Check status and logs: `docker compose ps` and `docker compose logs --tail=100`.
- **Port conflicts:** every published port must differ from the first instance.
- **Port forwarding not applied:** the forwarding command must call the qBittorrent Web UI port (`8091` here), with **Bypass authentication for clients on localhost** enabled.
- **Container name clash:** both stacks need unique `container_name` values (`gluetun_1`, `qbittorrent_1`).
- **A variable is empty or reported missing (`:?err`):** run the `config` command above to see which variable is not set in your env files.

## Related

- [Docker Stack Catalog](/docker-stacks)
- [Arr Download](/arr-download)
- [Arr Stack](/arr-stack)
- [Gluetun](/gluetun)

---
*Educational example. Review every path, port and credential before you deploy, and never publish your `.secret` file.*
