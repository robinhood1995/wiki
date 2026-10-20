---
title: Arr Download (Gluetun + qBittorrent + SABnzbd)
description: VPN-routed download clients: qBittorrent and SABnzbd behind Gluetun with Docker Compose.
published: true
date: 2026-10-18T03:27:28.468Z
tags: docker, docker-compose, homelab, downloads, gluetun, qbittorrent, sabnzbd, vpn
editor: markdown
dateCreated: 2026-10-18T03:26:52.529Z
---

# Arr Download (Gluetun + qBittorrent + SABnzbd)

> VPN-routed download clients: qBittorrent and SABnzbd behind Gluetun with Docker Compose.
{.is-info}

_Last updated: Oct 2026_

## Overview

This stack runs your download clients behind a VPN. Gluetun opens the VPN tunnel and owns the network, and qBittorrent (torrents) and SABnzbd (Usenet) join it with `network_mode: service:gluetun`. If the VPN drops, the clients lose internet access instead of leaking your real IP.

**Why you'd want it:** A kill switch by design, automatic port forwarding for better torrent speeds, and one place to manage VPN settings.

| Item | Value |
|---|---|
| Stack folder | `/opt/stacks/arr_download` |
| Category | Downloads |
| Main image | `lscr.io/linuxserver/qbittorrent:latest` |
| Services | 3 |
| Web UI | qBittorrent: http://<SERVER_IP>:8090, SABnzbd: http://<SERVER_IP>:8084 |
| Official docs | https://github.com/qdm12/gluetun-wiki |

## Requirements

- A Linux host with Docker Engine and the Docker Compose plugin (see [Docker](/docker))
- The shared folder layout and `.env`/`.secret` files from [Docker Folders](/dockerfolders) and [Environment](/environment)
- A VPN provider supported by Gluetun (this example uses ProtonVPN with port forwarding)
- `/dev/net/tun` available on the host
- Download folders on the same filesystem as your media for hardlinks (see [Docker Folders](/dockerfolders))

## Compose

Folder layout:

```text
/opt/stacks/
├── _shared/
│   ├── .env        # shared, non-secret settings
│   └── .secret     # passwords, tokens and keys (chmod 600, never commit)
└── arr_download/
    └── compose.yaml
/opt/appdata/arr_download/   # persistent application data
```

`/opt/stacks/arr_download/compose.yaml`:

```yaml
networks:
  media-network:
    name: media-network
    driver: bridge
services:
  gluetun:
    image: qmcgaw/gluetun:v3
    container_name: gluetun
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
      - VPN_PORT_FORWARDING_UP_COMMAND=/bin/sh -c 'wget -O- --retry-connrefused --post-data "json={\"listen_port\":{{PORTS}}}" http://127.0.0.1:8090/api/v2/app/setPreferences 2>&1'
      - SERVER_COUNTRIES=${SERVER_COUNTRIES:?err}
      - SHADOWSOCKS=on
      - FIREWALL_OUTBOUND_SUBNETS=<LAN_SUBNET> # <-- Your actual LAN subnet
      # Optional: Use Docker DNS for inter-container hostname resolution
      #- DNS_ADDRESS=127.0.0.11
    logging:
      driver: json-file
      options:
        max-file: ${DOCKERLOGGING_MAXFILE:?err}  # Max number of log files
        max-size: ${DOCKERLOGGING_MAXSIZE:?err}  # Max file size
    volumes:
      - ${DOCKERFOLDER:?err}/gluetun/config:/gluetun
    ports:
      - 6881:6881/tcp   # Transmission/Deluge: Torrent Port TCP
      - 6881:6881/udp   # Transmission/Deluge: Torrent Port UDP
      - 8084:8080       # WebUI Portal: SABnzbd
      - 8090:8090/tcp   # WebUI Portal: qBittorrent
      #- 8112:8112       # WebUI Portal: Deluge
      #- 9091:9091       # Transmission web UI
      #- 9090:9090       # SABnzbd: API port
      #- 51413:51413     # Transmission peer port (TCP)
      #- 51413:51413/udp # Transmission peer port (UDP)
      #- 58846:58846     # Deluge RPC
      - 8888:8888/tcp   # Gluetun Local Network HTTP proxy
      - 8388:8388/tcp   # Gluetun Local Network Shadowsocks
      - 8388:8388/udp   # Gluetun Local Network Shadowsocks
    networks:
      - media-network
    restart: unless-stopped
    sysctls:
      - net.ipv6.conf.all.disable_ipv6=1

  qbittorrent:
    image: lscr.io/linuxserver/qbittorrent:latest
    container_name: qbittorrent
    depends_on:
      gluetun:
        condition: service_healthy
    environment:
      - PUID=${PUID:?err}    # User ID
      - PGID=${PGID:?err}    # Group ID
      - TZ=${TZ:?err}        # TimeZone
      - UMASK=${UMASK:?err}  # File Umask
      - WEBUI_PORT=8090
    logging:
      driver: json-file
      options:
        max-file: ${DOCKERLOGGING_MAXFILE:?err}  # Max number of log files
        max-size: ${DOCKERLOGGING_MAXSIZE:?err}  # Max file size
    volumes:
      - ${DOCKERFOLDER:?err}/qbittorrent/config:/config
      - ${TORRENTDOWNLOAD:?err}:/data/torrents
      - ${MEDIAFOLDER:?err}:/data/media
    network_mode: "service:gluetun"
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

  sabnzbd:
    image: lscr.io/linuxserver/sabnzbd:latest
    container_name: sabnzbd
    depends_on:
      gluetun:
        condition: service_healthy
    environment:
      - PUID=${PUID:?err}    # User ID
      - PGID=${PGID:?err}    # Group ID
      - TZ=${TZ:?err}        # TimeZone
      - UMASK=${UMASK:?err}  # File Umask
      - DOCKER_MODS=ghcr.io/gilbn/theme.park:sabnzbd
      - TP_THEME=${TP_THEME:?err}
    logging:
      driver: json-file
      options:
        max-file: ${DOCKERLOGGING_MAXFILE:?err}  # Max number of log files
        max-size: ${DOCKERLOGGING_MAXSIZE:?err}  # Max file size
    volumes:
      - ${DOCKERFOLDER:?err}/sabnzbd:/config
      - ${USENETDOWNLOAD:?err}:/data/usenet
      - ${MEDIAFOLDER:?err}:/data/media
    network_mode: "service:gluetun"
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
TP_THEME=organizr
USENETDOWNLOAD=/srv/data/usenet
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
| `TP_THEME` | `.env` | theme.park theme name for SABnzbd | `organizr` |
| `USENETDOWNLOAD` | `.env` | Usenet download folder | `/srv/data/usenet` |

## Ports

| Host port | Container port | Protocol | Service | Purpose |
|---|---|---|---|---|
| 6881 | 6881 | tcp | `gluetun` | Transmission/Deluge: Torrent Port TCP |
| 6881 | 6881 | udp | `gluetun` | Transmission/Deluge: Torrent Port UDP |
| 8084 | 8080 | tcp | `gluetun` | WebUI Portal: SABnzbd |
| 8090 | 8090 | tcp | `gluetun` | WebUI Portal: qBittorrent |
| 8888 | 8888 | tcp | `gluetun` | Gluetun Local Network HTTP proxy |
| 8388 | 8388 | tcp | `gluetun` | Gluetun Local Network Shadowsocks |
| 8388 | 8388 | udp | `gluetun` | Gluetun Local Network Shadowsocks |

> Apps with `network_mode: service:<gluetun>` share the VPN container's network, so their ports are published on the Gluetun service.
{.is-info}

## Volumes

| Host path | Container path | Service | Purpose |
|---|---|---|---|
| `${DOCKERFOLDER}/gluetun/config` | `/gluetun` | `gluetun` | Application configuration |
| `${DOCKERFOLDER}/qbittorrent/config` | `/config` | `qbittorrent` | Application configuration |
| `${TORRENTDOWNLOAD}` | `/data/torrents` | `qbittorrent` | Downloads |
| `${MEDIAFOLDER}` | `/data/media` | `qbittorrent` | Media library |
| `${DOCKERFOLDER}/sabnzbd` | `/config` | `sabnzbd` | Application configuration |
| `${USENETDOWNLOAD}` | `/data/usenet` | `sabnzbd` | Downloads |
| `${MEDIAFOLDER}` | `/data/media` | `sabnzbd` | Media library |

## Resource limits

Recommended limits, already set in the compose file under `deploy.resources`. Raise them if the app is starved, or lower them on small hosts.

| Service | CPU limit | Memory limit | Memory reservation | GPU |
|---|---|---|---|---|
| `qbittorrent` | 2.0 | 2000M | - | - |
| `sabnzbd` | 2.0 | 2000M | - | - |

## Setup

1. Create the folders:
   ```bash
   sudo mkdir -p /opt/stacks/arr_download /opt/appdata/arr_download
   sudo chown -R $USER: /opt/stacks/arr_download
   ```
2. Add the variables above to `/opt/stacks/_shared/.env` and `/opt/stacks/_shared/.secret` (`chmod 600 .secret`).
3. Save the compose file as `/opt/stacks/arr_download/compose.yaml`.
4. Check the configuration, then start the stack:
   ```bash
   cd /opt/stacks/arr_download
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret config --quiet
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
   ```
5. Watch the first start: `docker compose logs -f`.
6. Wait for Gluetun to report healthy: `docker logs gluetun`.
7. Check the VPN exit IP: `docker exec gluetun wget -qO- https://ipinfo.io/ip`. It must not be your home IP.
8. Open qBittorrent on port 8090. The temporary admin password is printed in `docker logs qbittorrent`. Change it right away.
9. Open SABnzbd on port 8084 and complete the wizard with your Usenet provider.
10. Set the download paths to `/data/torrents` and `/data/usenet`.

> Tip: create a small wrapper such as `alias dc='docker compose --env-file /opt/stacks/_shared/.env --env-file /opt/stacks/_shared/.secret'` so every stack uses the same env files.
{.is-success}

## Updating

```bash
cd /opt/stacks/arr_download
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret pull
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
docker image prune -f
```

Some images here are pinned to a version. Read the upstream release notes before changing the tag.

Back up `/opt/appdata/arr_download` before major upgrades. [Watchtower](/watchtower) can automate updates for stateless apps.

## Troubleshooting

- Check status and logs: `docker compose ps` and `docker compose logs --tail=100`.
- **Clients have no internet:** check Gluetun first; every client shares its network.
- **Cannot reach the web UIs from the LAN:** set `FIREWALL_OUTBOUND_SUBNETS` to your LAN subnet, for example `192.168.1.0/24`.
- **Port forwarding not applied:** `VPN_PORT_FORWARDING_UP_COMMAND` pushes the forwarded port to the qBittorrent API, so the URL must use the qBittorrent Web UI port (`8090` here). In qBittorrent, enable **Bypass authentication for clients on localhost**.
- **ProtonVPN:** the `+pmp` suffix on the OpenVPN user enables port forwarding.
- **A variable is empty or reported missing (`:?err`):** run the `config` command above to see which variable is not set in your env files.

## Related

- [Docker Stack Catalog](/docker-stacks)
- [Arr Stack](/arr-stack)
- [Arr Download 1](/arr-download-1)
- [Gluetun](/gluetun)
- [SABnzbd](/sabnzdb)

---
*Educational example. Review every path, port and credential before you deploy, and never publish your `.secret` file.*
