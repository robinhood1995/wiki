---
title: Arr Stack (media automation suite)
description: Complete *arr media automation suite (Sonarr, Radarr, Lidarr, Prowlarr, Bazarr and more) behind Gluetun, plus Wizarr.
published: true
date: 2026-10-18T03:27:34.055Z
tags: docker, docker-compose, downloads, gluetun, homelab, media, vpn, arr, prowlarr, radarr, sonarr, wizarr
editor: markdown
dateCreated: 2026-10-18T03:26:52.568Z
---

# Arr Stack (media automation suite)

> Complete *arr media automation suite (Sonarr, Radarr, Lidarr, Prowlarr, Bazarr and more) behind Gluetun, plus Wizarr.
{.is-info}

_Last updated: Oct 2026_

## Overview

The arr stack automates a media library. Prowlarr manages indexers, Sonarr (TV), Radarr (movies), Lidarr (music), Readarr/LazyLibrarian (books) and Mylar3 (comics) search and grab releases, Bazarr fetches subtitles, and Unpackerr extracts archives. Helpers include FlareSolverr (Cloudflare challenges), Profilarr (quality profiles), qbitrr and Cleanuparr (stalled download cleanup), qui (multi-qBittorrent UI), Calibre and rreading-glasses (book metadata). Wizarr handles user invitations for your media server. Every app shares one Gluetun VPN network.

**Why you'd want it:** Request a show once and it arrives in your library, correctly named, with subtitles, without you lifting a finger.

| Item | Value |
|---|---|
| Stack folder | `/opt/stacks/arr_stack` |
| Category | Downloads |
| Main image | `lscr.io/linuxserver/sonarr:latest` |
| Services | 18 |
| Web UI | Sonarr :8989, Radarr :7878, Prowlarr :9696, Lidarr :8686, Bazarr :6767, Wizarr :5690 (all on <SERVER_IP>) |
| Official docs | https://wiki.servarr.com/ |

> **Wizarr** is documented here because it runs inside this stack. A standalone Wizarr stack also exists; see [Wizarr](/Wizarr) if you prefer to run it on its own. Run only one of them, because both use port 5690.
{.is-info}

## Requirements

- A Linux host with Docker Engine and the Docker Compose plugin (see [Docker](/docker))
- The shared folder layout and `.env`/`.secret` files from [Docker Folders](/dockerfolders) and [Environment](/environment)
- A VPN provider supported by Gluetun
- A download client, e.g. [Arr Download](/arr-download)
- One shared data root (`SERVAAR`) holding both downloads and media so hardlinks work
- PostgreSQL only if you run rreading-glasses

## Compose

Folder layout:

```text
/opt/stacks/
├── _shared/
│   ├── .env        # shared, non-secret settings
│   └── .secret     # passwords, tokens and keys (chmod 600, never commit)
└── arr_stack/
    └── compose.yaml
/opt/appdata/arr_stack/   # persistent application data
```

`/opt/stacks/arr_stack/compose.yaml`:

```yaml
networks:
  media-network:
    name: media-network
    driver: bridge
services:
  gluetun_0:
    image: qmcgaw/gluetun:v3
    container_name: gluetun_0
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
      - VPN_PORT_FORWARDING_UP_COMMAND=/bin/sh -c 'wget -O- --retry-connrefused --post-data "json={\"listen_port\":{{PORTS}}}" http://127.0.0.1:8080/api/v2/app/setPreferences 2>&1'
      - SERVER_COUNTRIES=${SERVER_COUNTRIES:?err}
      - SHADOWSOCKS=on
      # Your VPN credentials and settings here (e.g., VPN_SERVICE_PROVIDER, VPN_TYPE)
      - FIREWALL_OUTBOUND_SUBNETS=<LAN_SUBNET> # <-- Your actual LAN subnet
      # Optional: Use Docker DNS for inter-container hostname resolution
      - DNS_ADDRESS=127.0.0.11
    logging:
      driver: json-file
      options:
        max-file: ${DOCKERLOGGING_MAXFILE:?err}  # Max number of log files
        max-size: ${DOCKERLOGGING_MAXSIZE:?err}  # Max file size
    volumes:
      - ${DOCKERFOLDER:?err}/gluetun_0/config:/gluetun
    ports:
      - 8890:8890/tcp   # Local Network HTTP proxy: Gluetun
      - 8390:8390/tcp   # Local Network Shadowsocks: Gluetun
      - 8390:8390/udp   # Local Network Shadowsocks: Gluetun
      - 7476:7476       # WebUI Portal: qui
      - 9696:9696 # Prowlarr web interface port
      - 8191:8191 # FlareSolverr web interface port
      - 7878:7878 # Radarr web interface port
      - 8989:8989 # Sonarr web interface port
      - 6767:6767 # Bazarr web interface port
      - 6970:6969 # qBitrr web interface port
      - 8686:8686 # Lidarr web interface port
      - 5299:5299 # LazyLibrarian web interface port
      - 8078:8090 # Mylar3 web interface port
      - 6969:6969 # Whisparr web interface port
      - 11011:11011 # Cleanuparr web interface port
      - 8787:8787 # Port for Readarr web UI
      - 6868:6868 # Profilarr Web UI
      - 8079:8080 # Calibre GUI access (optional password via environment variable)
      - 8181:8181 # Calibre GUI access (HTTPS, self-signed cert)
      - 8081:8081 # Calibre's built-in webserver (enable in Calibre preferences)
      - 8788:8788 # Reading-Glasses GUI UI
      - 5690:5690 # Wizarr WEB GUI
#      - 8092:8000 # MediaManager web interface port
#      - 8093:3000 # Cinephage web interface port
    networks:
      - media-network
    restart: unless-stopped
    sysctls:
      - net.ipv6.conf.all.disable_ipv6=1

  bazarr:
    image: ghcr.io/hotio/bazarr:nightly
    container_name: bazarr
    environment:
      - PUID=${PUID}    # User ID
      - PGID=${PGID}    # Group ID
      - TZ=${TZ}        # TimeZone
      - UMASK=${UMASK}  # File Umask
      - VPN_ENABLED=True
    logging:
      driver: json-file
      options:
        max-file: ${DOCKERLOGGING_MAXFILE}
        max-size: ${DOCKERLOGGING_MAXSIZE}
#    ports:
#      - 6767:6767 # Bazarr web interface port
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - ${DOCKERFOLDER}/bazarr:/config
      - ${SERVAAR}:/data
    network_mode: "service:gluetun_0"
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 1000M

  calibre:
    image: lscr.io/linuxserver/calibre:latest
    container_name: calibre
    environment:
      - PUID=${PUID}    # User ID
      - PGID=${PGID}    # Group ID
      - TZ=${TZ}        # TimeZone
      - UMASK=${UMASK}  # File Umask
      - CUSTOM_USER=admin
      - PASSWORD=<CALIBRE_PASSWORD> # Optional: Set a password for the GUI
      - CLI_ARGS= # Optional: Pass CLI arguments to Calibre
      - CUSTOM_PORT=8078
    volumes:
      - ${DOCKERFOLDER}/calibre/config:/config # Calibre database and library
      - ${MEDIAFOLDER}/books:/books # Optional: Mount a directory for books to be added
#    ports:
#      - 8078:8080 # Calibre GUI access (optional password via environment variable)
#      - 8181:8181 # Calibre GUI access (HTTPS, self-signed cert)
#      - 8081:8081 # Calibre's built-in webserver (enable in Calibre preferences)
    security_opt:
      - seccomp=unconfined # May be required for GUI apps in Docker
    network_mode: "service:gluetun_0"
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 2000M

  cleanuparr:
    image: ghcr.io/cleanuparr/cleanuparr:latest # Or a specific version
    container_name: cleanuparr
    environment:
      - PUID=${PUID}    # User ID
      - PGID=${PGID}    # Group ID
      - TZ=${TZ}        # TimeZone
      - UMASK=${UMASK}  # File Umask
#    ports:
#      - 11011:11011 # Cleanuparr web interface port
    volumes:
      - ${DOCKERFOLDER}/cleanuparr/config:/config # Whisparr configuration files
      - ${SERVAAR}:/data
    network_mode: "service:gluetun_0"
    restart: unless-stopped

  flaresolverr:
    image: flaresolverr/flaresolverr:latest
    container_name: flaresolverr
    environment:
      - LOG_LEVEL=info
      - CAPTCHA_SOLVER=none
      - PUID=${PUID}    # User ID
      - PGID=${PGID}    # Group ID
      - TZ=${TZ}        # TimeZone
      - UMASK=${UMASK}  # File Umask
    logging:
      driver: json-file
      options:
        max-file: ${DOCKERLOGGING_MAXFILE}  # Max number of log files
        max-size: ${DOCKERLOGGING_MAXSIZE}  # Max file size
#    ports:
#      - 8191:8191 # FlareSolverr web interface port
    volumes:
      - ${DOCKERFOLDER}/flaresolverr/config:/config
    network_mode: "service:gluetun_0"
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 1000M

  lazylibrarian:
    image: lscr.io/linuxserver/lazylibrarian:latest
    container_name: lazylibrarian
    environment:
      - PUID=${PUID}    # User ID
      - PGID=${PGID}    # Group ID
      - TZ=${TZ}        # TimeZone
      - UMASK=${UMASK}  # File Umask
    logging:
      driver: json-file
      options:
        max-file: ${DOCKERLOGGING_MAXFILE}
        max-size: ${DOCKERLOGGING_MAXSIZE}
#    ports:
#      - 5299:5299 # LazyLibrarian web interface port
    volumes:
      - ${DOCKERFOLDER}/lazylibrarian/config:/config # Path to store configuration files
      - ${SERVAAR}:/data
      - ${MEDIAFOLDER}/books:/books
    network_mode: "service:gluetun_0"
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 1000M

  lidarr:
    image: lscr.io/linuxserver/lidarr:latest
    container_name: lidarr
    environment:
      - PUID=${PUID}    # User ID
      - PGID=${PGID}    # Group ID
      - TZ=${TZ}        # TimeZone
      - UMASK=${UMASK}  # File Umask
      - APIKEY=${LIDARR_API_KEY}
    logging:
      driver: json-file
      options:
        max-file: ${DOCKERLOGGING_MAXFILE}
        max-size: ${DOCKERLOGGING_MAXSIZE}
#    ports:
#      - 8686:8686 # Lidarr web interface port
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - ${DOCKERFOLDER}/lidarr/config:/config # Replace with your config path
      - ${SERVAAR}:/data
    depends_on:
      - prowlarr
    network_mode: "service:gluetun_0"
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 1000M

  mylar3:
    image: linuxserver/mylar3
    container_name: mylar3
    environment:
      - PUID=${PUID}    # User ID
      - PGID=${PGID}    # Group ID
      - TZ=${TZ}        # TimeZone
      - UMASK=${UMASK}  # File Umask
      - COMICVIN_BASEURL=#optional
      - KEY=${MYLAR_API_KEY}
    logging:
      driver: json-file
      options:
        max-file: ${DOCKERLOGGING_MAXFILE}
        max-size: ${DOCKERLOGGING_MAXSIZE}
#    ports:
#      - 8090:8090 # Mylar3 web interface port
    volumes:
      - ${DOCKERFOLDER}/mylar3/config:/config # Mylar3 configuration files
      - ${SERVAAR}:/data
      - ${MEDIAFOLDER}/comics:/comics
    depends_on:
      - prowlarr
    network_mode: "service:gluetun_0"
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 1000M

    #ports:
      #- 9705:9705
#    volumes:
#      - ${DOCKERFOLDER}/newtarr/config:/config
#    network_mode: "service:gluetun_0"
#    restart: unless-stopped

  profilarr:
    image: santiagosayshey/profilarr:latest # Use :beta for early access to new features
    container_name: profilarr
    environment:
      - PUID=${PUID}    # User ID
      - PGID=${PGID}    # Group ID
      - TZ=${TZ}        # TimeZone
      - UMASK=${UMASK}  # File Umask
      - LOG_LEVEL=debug
    logging:
      driver: json-file
      options:
        max-file: ${DOCKERLOGGING_MAXFILE}
        max-size: ${DOCKERLOGGING_MAXSIZE}
#    ports:
#      - 6868:6868 # Profilarr Web UI
    volumes:
      - ${DOCKERFOLDER}/profilarr/config:/config # Change to your desired config path
    network_mode: "service:gluetun_0"
    restart: unless-stopped

  prowlarr:
    image: lscr.io/linuxserver/prowlarr:latest
    container_name: prowlarr
    environment:
      - PUID=${PUID}    # User ID
      - PGID=${PGID}    # Group ID
      - TZ=${TZ}        # TimeZone
      - UMASK=${UMASK}  # File Umask
      - PROWLARR_API_KEY=${PROWLARR_API_KEY}
      - ARGS=
    logging:
      driver: json-file
      options:
        max-file: ${DOCKERLOGGING_MAXFILE}  # Max number of log files
        max-size: ${DOCKERLOGGING_MAXSIZE}  # Max file size
#    ports:
#      - 9696:9696 # Prowlarr web interface port
    volumes:
      - ${DOCKERFOLDER}/prowlarr/config:/config:rw # Host path for Prowlarr configuration
    depends_on:
      gluetun_0:
        condition: service_healthy
    network_mode: "service:gluetun_0"
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 1000M

  qbitrr:
    image: feramance/qbitrr:latest
    container_name: qbitrr
    tty: true
    environment:
      - PUID=${PUID}    # User ID
      - PGID=${PGID}    # Group ID
      - TZ=${TZ}        # TimeZone
      - UMASK=${UMASK}  # File Umask
    logging:
      driver: json-file
      options:
        max-file: ${DOCKERLOGGING_MAXFILE}  # Max number of log files
        max-size: ${DOCKERLOGGING_MAXSIZE}  # Max file size
    #ports:
      #- "6969:6969"
    volumes:
      - ${DOCKERFOLDER}/qbitrr/config:/config
      - ${SERVAAR}:/completed_downloads:rw
    network_mode: "service:gluetun_0"
    restart: unless-stopped
  qui:
    image: ghcr.io/autobrr/qui:latest
    container_name: qui
    environment:
      - PUID=${PUID:?err}    # User ID
      - PGID=${PGID:?err}    # Group ID
      - TZ=${TZ:?err}        # TimeZone
      - UMASK=${UMASK:?err}  # File Umask
    logging:
      driver: json-file
      options:
        max-file: ${DOCKERLOGGING_MAXFILE:?err}  # Max number of log files
        max-size: ${DOCKERLOGGING_MAXSIZE:?err}  # Max file size
    volumes:
      - ${DOCKERFOLDER:?err}/qui/config:/config
    network_mode: "service:gluetun_0"
    depends_on:
      gluetun_0:
        condition: service_healthy
    restart: unless-stopped

  radarr:
    image: lscr.io/linuxserver/radarr:latest
    container_name: radarr
    environment:
      - PUID=${PUID}    # User ID
      - PGID=${PGID}    # Group ID
      - TZ=${TZ}        # TimeZone
      - UMASK=${UMASK}  # File Umask
      - RADARR_API_KEY=${RADARR_API_KEY}
    logging:
      driver: json-file
      options:
        max-file: ${DOCKERLOGGING_MAXFILE}  # Max number of log files
        max-size: ${DOCKERLOGGING_MAXSIZE}  # Max file size
#    ports:
#      - 7878:7878 # Radarr web interface port
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - ${DOCKERFOLDER}/radarr/config:/config # Radarr configuration files
      - ${SERVAAR}:/data
    depends_on:
      - prowlarr
    network_mode: "service:gluetun_0"
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 1000M

  readarr:
    image: lscr.io/linuxserver/readarr:0.4.19-nightly
    container_name: readarr
    environment:
      - PUID=${PUID}    # User ID
      - PGID=${PGID}    # Group ID
      - TZ=${TZ}        # TimeZone
      - UMASK=${UMASK}  # File Umask
#    ports:
#      - 8787:8787 # Port for Readarr web UI
    volumes:
      - ${DOCKERFOLDER}/readarr/config:/config # Whisparr configuration files
      - ${SERVAAR}:/data
    network_mode: "service:gluetun_0"
    depends_on:
      - prowlarr
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 1000M

  rreading-glasses:
    image: blampe/rreading-glasses:latest
    pull_policy: always
    container_name: rreading-glasses

    entrypoint: ["/main", "serve"]
    command:
      - --upstream=www.goodreads.com
      - --verbose
    mem_limit: 128m
    environment:
      - POSTGRES_HOST=${DB_POSTGRES_HOST:?err}
      - POSTGRES_DATABASE=rreading_glasses
      - POSTGRES_USER=rreading_glasses
      - POSTGRES_PASSWORD=<RG_DB_PASSWORD>
      #- POSTGRES_PORT=5432
#    ports:
#      - 8788:8788
    network_mode: "service:gluetun_0"
    depends_on:
      - prowlarr
    restart: unless-stopped

  sonarr:
    image: lscr.io/linuxserver/sonarr:latest
    container_name: sonarr
    environment:
      - PUID=${PUID}    # User ID
      - PGID=${PGID}    # Group ID
      - TZ=${TZ}        # TimeZone
      - UMASK=${UMASK}  # File Umask
      - SONARR__API=${SONARR_API_KEY}
    logging:
      driver: json-file
      options:
        max-file: ${DOCKERLOGGING_MAXFILE}  # Max number of log files
        max-size: ${DOCKERLOGGING_MAXSIZE}  # Max file size
#    ports:
#      - 8989:8989 # Sonarr web interface port
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - ${DOCKERFOLDER}/sonarr/config:/config # Replace with your desired config path
      - ${SERVAAR}:/data
    depends_on:
      - prowlarr
    network_mode: "service:gluetun_0"
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 1000M

  unpackerr:
    image: golift/unpackerr:latest
    container_name: unpackerr
    environment:
      - PUID=${PUID}    # User ID
      - PGID=${PGID}    # Group ID
      - TZ=${TZ}        # TimeZone
      - UMASK=${UMASK}  # File Umask
# Documentation on all Environment Variables can be found at:
# https://github.com/davidnewhall/unpackerr#docker-env-variables
      - UN_DEBUG=false
      - UN_LOG_FILE=
      - UN_LOG_FILES=10
      - UN_LOG_FILE_MB=10
      - UN_INTERVAL=2m
      - UN_START_DELAY=1m
      - UN_RETRY_DELAY=5m
      - UN_MAX_RETRIES=3
      - UN_PARALLEL=1
      - UN_FILE_MODE=0664
      - UN_DIR_MODE=0775
      # Sonarr Config - Copy API Key from: http://sonarr:8989/general/settings
      - UN_SONARR_0_URL=http://<SERVER_IP>:8989
      - UN_SONARR_0_API_KEY=${SONARR_API_KEY}
      - UN_SONARR_0_PATHS_0=${SERVAAR}/torrents/incomplete/tv-sonarr
      #- UN_SONARR_0_PATHS_1=/data/torrents/series
      - UN_SONARR_0_PROTOCOLS=torrent
      - UN_SONARR_0_TIMEOUT=10s
      - UN_SONARR_0_DELETE_ORIG=false
      - UN_SONARR_0_DELETE_DELAY=5m
      # Radarr Config - Copy API Key from: http://radarr:7878/general/settings
      - UN_RADARR_0_URL=http://<SERVER_IP>:7878
      - UN_RADARR_0_API_KEY=${RADARR_API_KEY}
      - UN_RADARR_0_PATHS_0=${SERVAAR}/torrents/incomplete/Movies
      - UN_RADARR_0_PATHS_1=${SERVAAR}/torrents/incomplete/radarr
      - UN_RADARR_0_PROTOCOLS=torrent
      - UN_RADARR_0_TIMEOUT=10s
      - UN_RADARR_0_DELETE_ORIG=false
      - UN_RADARR_0_DELETE_DELAY=5m
      # Lidarr Config - Copy API Key from: http://lidarr:8686/general/settings
      - UN_LIDARR_0_URL=http://<SERVER_IP>:8686
      - UN_LIDARR_0_API_KEY=${LIDARR_API_KEY}
      - UN_LIDARR_0_PATHS_0=${SERVAAR}/torrents/incomplete/lidarr
      - UN_LIDARR_0_PROTOCOLS=torrent
      - UN_LIDARR_0_TIMEOUT=10s
      - UN_LIDARR_0_DELETE_ORIG=false
      - UN_LIDARR_0_DELETE_DELAY=15m
    volumes:
      - ${DOCKERFOLDER}/unpackerr:/config
      - ${SERVAAR}:/data
    network_mode: "service:gluetun_0"
    security_opt:
      - no-new-privileges:true
    restart: unless-stopped

  wizarr:
    image: ghcr.io/wizarrrr/wizarr:latest
    container_name: wizarr
    environment:
      - APP_URL=https://wizarr.<YOUR_DOMAIN>
      - PUID=${PUID}    # User ID
      - PGID=${PGID}    # Group ID
      - TZ=${TZ}        # TimeZone
      - UMASK=${UMASK}  # File Umask
#    ports:
#      - 5690:5690 # Wizarr WEB GUI
    network_mode: "service:gluetun_0"
    volumes:
      - ${DOCKERFOLDER}/wizarr/database:/data/database
      - ${DOCKERFOLDER}/wizarr/steps:/data/wizard_steps
    restart: unless-stopped
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
SERVAAR=/srv/data
MEDIAFOLDER=/srv/media
DB_POSTGRES_HOST=<DB_HOST>
```

`/opt/stacks/_shared/.secret`:

```ini
OPENVPN_USERNAME=<VPN_USERNAME>
OPENVPN_PASSWORD=<VPN_PASSWORD>
WIREGUARD_PRIVATE_KEY=<WIREGUARD_PRIVATE_KEY>
LIDARR_API_KEY=<LIDARR_API_KEY>
MYLAR_API_KEY=<MYLAR_API_KEY>
PROWLARR_API_KEY=<PROWLARR_API_KEY>
RADARR_API_KEY=<RADARR_API_KEY>
SONARR_API_KEY=<SONARR_API_KEY>
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
| `SERVAAR` | `.env` | Shared data root mounted as /data in every *arr app (downloads + media on one filesystem for hardlinks) | `/srv/data` |
| `MEDIAFOLDER` | `.env` | Root of your media library (movies, tv, music, books) | `/srv/media` |
| `LIDARR_API_KEY` | `.secret` | Lidarr API key (Settings > General) | `<LIDARR_API_KEY>` |
| `MYLAR_API_KEY` | `.secret` | Mylar3 API key | `<MYLAR_API_KEY>` |
| `PROWLARR_API_KEY` | `.secret` | Prowlarr API key (Settings > General) | `<PROWLARR_API_KEY>` |
| `RADARR_API_KEY` | `.secret` | Radarr API key (Settings > General) | `<RADARR_API_KEY>` |
| `DB_POSTGRES_HOST` | `.env` | Hostname or IP of your PostgreSQL server | `<DB_HOST>` |
| `SONARR_API_KEY` | `.secret` | Sonarr API key (Settings > General) | `<SONARR_API_KEY>` |

## Ports

| Host port | Container port | Protocol | Service | Purpose |
|---|---|---|---|---|
| 8890 | 8890 | tcp | `gluetun_0` | Local Network HTTP proxy: Gluetun |
| 8390 | 8390 | tcp | `gluetun_0` | Local Network Shadowsocks: Gluetun |
| 8390 | 8390 | udp | `gluetun_0` | Local Network Shadowsocks: Gluetun |
| 7476 | 7476 | tcp | `gluetun_0` | WebUI Portal: qui |
| 9696 | 9696 | tcp | `gluetun_0` | Prowlarr web interface port |
| 8191 | 8191 | tcp | `gluetun_0` | FlareSolverr web interface port |
| 7878 | 7878 | tcp | `gluetun_0` | Radarr web interface port |
| 8989 | 8989 | tcp | `gluetun_0` | Sonarr web interface port |
| 6767 | 6767 | tcp | `gluetun_0` | Bazarr web interface port |
| 6970 | 6969 | tcp | `gluetun_0` | qBitrr web interface port |
| 8686 | 8686 | tcp | `gluetun_0` | Lidarr web interface port |
| 5299 | 5299 | tcp | `gluetun_0` | LazyLibrarian web interface port |
| 8078 | 8090 | tcp | `gluetun_0` | Mylar3 web interface port |
| 6969 | 6969 | tcp | `gluetun_0` | Whisparr web interface port |
| 11011 | 11011 | tcp | `gluetun_0` | Cleanuparr web interface port |
| 8787 | 8787 | tcp | `gluetun_0` | Port for Readarr web UI |
| 6868 | 6868 | tcp | `gluetun_0` | Profilarr Web UI |
| 8079 | 8080 | tcp | `gluetun_0` | Calibre GUI access (optional password via environment variable) |
| 8181 | 8181 | tcp | `gluetun_0` | Calibre GUI access (HTTPS, self-signed cert) |
| 8081 | 8081 | tcp | `gluetun_0` | Calibre's built-in webserver (enable in Calibre preferences) |
| 8788 | 8788 | tcp | `gluetun_0` | Reading-Glasses GUI UI |
| 5690 | 5690 | tcp | `gluetun_0` | Wizarr WEB GUI |

> Apps with `network_mode: service:<gluetun>` share the VPN container's network, so their ports are published on the Gluetun service.
{.is-info}

## Volumes

| Host path | Container path | Service | Purpose |
|---|---|---|---|
| `${DOCKERFOLDER}/gluetun_0/config` | `/gluetun` | `gluetun_0` | Application configuration |
| `/etc/localtime` | `/etc/localtime` | `bazarr` | Host time zone (read-only) |
| `${DOCKERFOLDER}/bazarr` | `/config` | `bazarr` | Application configuration |
| `${SERVAAR}` | `/data` | `bazarr` | Application data |
| `${DOCKERFOLDER}/calibre/config` | `/config` | `calibre` | Calibre database and library |
| `${MEDIAFOLDER}/books` | `/books` | `calibre` | Optional: Mount a directory for books to be added |
| `${DOCKERFOLDER}/cleanuparr/config` | `/config` | `cleanuparr` | Whisparr configuration files |
| `${SERVAAR}` | `/data` | `cleanuparr` | Application data |
| `${DOCKERFOLDER}/flaresolverr/config` | `/config` | `flaresolverr` | Application configuration |
| `${DOCKERFOLDER}/lazylibrarian/config` | `/config` | `lazylibrarian` | Path to store configuration files |
| `${SERVAAR}` | `/data` | `lazylibrarian` | Application data |
| `${MEDIAFOLDER}/books` | `/books` | `lazylibrarian` | Optional: Mount a directory for books to be added |
| `/etc/localtime` | `/etc/localtime` | `lidarr` | Host time zone (read-only) |
| `${DOCKERFOLDER}/lidarr/config` | `/config` | `lidarr` | Replace with your config path |
| `${SERVAAR}` | `/data` | `lidarr` | Application data |
| `${DOCKERFOLDER}/mylar3/config` | `/config` | `mylar3` | Mylar3 configuration files |
| `${SERVAAR}` | `/data` | `mylar3` | Application data |
| `${MEDIAFOLDER}/comics` | `/comics` | `mylar3` | Persistent data |
| `${DOCKERFOLDER}/profilarr/config` | `/config` | `profilarr` | Change to your desired config path |
| `${DOCKERFOLDER}/prowlarr/config` | `/config` | `prowlarr` | Host path for Prowlarr configuration |
| `${DOCKERFOLDER}/qbitrr/config` | `/config` | `qbitrr` | Application configuration |
| `${SERVAAR}` | `/completed_downloads` | `qbitrr` | Downloads |
| `${DOCKERFOLDER}/qui/config` | `/config` | `qui` | Application configuration |
| `/etc/localtime` | `/etc/localtime` | `radarr` | Host time zone (read-only) |
| `${DOCKERFOLDER}/radarr/config` | `/config` | `radarr` | Radarr configuration files |
| `${SERVAAR}` | `/data` | `radarr` | Application data |
| `${DOCKERFOLDER}/readarr/config` | `/config` | `readarr` | Whisparr configuration files |
| `${SERVAAR}` | `/data` | `readarr` | Application data |
| `/etc/localtime` | `/etc/localtime` | `sonarr` | Host time zone (read-only) |
| `${DOCKERFOLDER}/sonarr/config` | `/config` | `sonarr` | Replace with your desired config path |
| `${SERVAAR}` | `/data` | `sonarr` | Application data |
| `${DOCKERFOLDER}/unpackerr` | `/config` | `unpackerr` | Application configuration |
| `${SERVAAR}` | `/data` | `unpackerr` | Application data |
| `${DOCKERFOLDER}/wizarr/database` | `/data/database` | `wizarr` | Application data |
| `${DOCKERFOLDER}/wizarr/steps` | `/data/wizard_steps` | `wizarr` | Application data |

## Resource limits

Recommended limits, already set in the compose file under `deploy.resources`. Raise them if the app is starved, or lower them on small hosts.

| Service | CPU limit | Memory limit | Memory reservation | GPU |
|---|---|---|---|---|
| `bazarr` | 1.0 | 1000M | - | - |
| `calibre` | 2.0 | 2000M | - | - |
| `flaresolverr` | 1.0 | 1000M | - | - |
| `lazylibrarian` | 1.0 | 1000M | - | - |
| `lidarr` | 1.0 | 1000M | - | - |
| `mylar3` | 1.0 | 1000M | - | - |
| `prowlarr` | 1.0 | 1000M | - | - |
| `radarr` | 1.0 | 1000M | - | - |
| `readarr` | 1.0 | 1000M | - | - |
| `rreading-glasses` | - | 128m | - | - |
| `sonarr` | 1.0 | 1000M | - | - |

## Setup

1. Create the folders:
   ```bash
   sudo mkdir -p /opt/stacks/arr_stack /opt/appdata/arr_stack
   sudo chown -R $USER: /opt/stacks/arr_stack
   ```
2. Add the variables above to `/opt/stacks/_shared/.env` and `/opt/stacks/_shared/.secret` (`chmod 600 .secret`).
3. Save the compose file as `/opt/stacks/arr_stack/compose.yaml`.
4. Check the configuration, then start the stack:
   ```bash
   cd /opt/stacks/arr_stack
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret config --quiet
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
   ```
5. Watch the first start: `docker compose logs -f`.
6. Start the stack and confirm Gluetun is healthy: `docker logs gluetun_0`.
7. Open Prowlarr (:9696), add indexers, then add Sonarr, Radarr and Lidarr under **Settings > Apps**.
8. In each *arr app, add your download client and set root folders under `/data/media/...`.
9. Copy each app's API key into `.secret` so Unpackerr and other helpers can connect, then restart the stack.
10. Open Wizarr (:5690), connect your Plex or Jellyfin server and create your first invitation.

> Tip: create a small wrapper such as `alias dc='docker compose --env-file /opt/stacks/_shared/.env --env-file /opt/stacks/_shared/.secret'` so every stack uses the same env files.
{.is-success}

## Updating

```bash
cd /opt/stacks/arr_stack
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret pull
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
docker image prune -f
```

Some images here are pinned to a version. Read the upstream release notes before changing the tag.

Back up `/opt/appdata/arr_stack` before major upgrades. [Watchtower](/watchtower) can automate updates for stateless apps.

## Troubleshooting

- Check status and logs: `docker compose ps` and `docker compose logs --tail=100`.
- **Apps can't talk to each other:** inside the shared Gluetun network every app is `localhost`, e.g. Sonarr reaches Prowlarr at `http://localhost:9696`.
- **Copies instead of hardlinks:** downloads and media must be in the same `/data` mount.
- **Web UI unreachable from the LAN:** set `FIREWALL_OUTBOUND_SUBNETS` to your LAN subnet.
- **`VPN_PORT_FORWARDING_UP_COMMAND`:** there is no qBittorrent in this stack, so the command does nothing here. Remove it, or point it at a client that shares this Gluetun network.
- **Readarr metadata errors:** Readarr is retired upstream; point it at rreading-glasses for metadata or use LazyLibrarian.
- **A variable is empty or reported missing (`:?err`):** run the `config` command above to see which variable is not set in your env files.

## Related

- [Docker Stack Catalog](/docker-stacks)
- [Arr Download](/arr-download)
- [Seerr](/seerr)
- [Wizarr](/Wizarr)
- [Gluetun](/gluetun)
- [Plex](/plex)
- [Jellyfin](/jellyfin)

---
*Educational example. Review every path, port and credential before you deploy, and never publish your `.secret` file.*
