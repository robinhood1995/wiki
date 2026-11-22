---
title: Docker Stack Catalog
description: Catalog of 54 homelab Docker Compose stacks grouped by category, with links to step-by-step guides.
published: true
date: 2026-11-22T06:33:38.849Z
tags: catalog, docker, docker-compose, homelab
editor: markdown
dateCreated: 2026-09-08T21:11:08.018Z
---

# Docker Stack Catalog

> Ready-to-use Docker Compose stacks for your homelab, grouped by what they do. Every guide follows the same structure, so it's easy to follow by hand or parse with a script.
{.is-info}

_Last updated: Nov 2026_

## How these guides work

Every stack page has the same sections in the same order: **Overview, Requirements, Compose, Environment variables, Ports, Volumes, Resource limits, Setup, Updating, Troubleshooting, Related**. Compose blocks are complete and copy-pasteable. Values you must change are written as `<PLACEHOLDER>`, for example `<SERVER_IP>`, `<YOUR_DOMAIN>`, `<TZ>` or `<DB_PASSWORD>`.

### Conventions used on every page

| Convention | Value |
|---|---|
| Compose files | `/opt/stacks/<name>/compose.yaml` |
| Persistent app data | `/opt/appdata/<name>/` |
| Media and downloads | `/srv/media`, `/srv/data` |
| Shared settings | `/opt/stacks/_shared/.env` |
| Shared secrets | `/opt/stacks/_shared/.secret` (chmod 600, never commit) |
| Start a stack | `docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d` |
| Log rotation | `DOCKERLOGGING_MAXFILE` / `DOCKERLOGGING_MAXSIZE` on every service |

### Start here

1. [Install Docker](/docker)
2. [Docker Folders](/dockerfolders): create the folder layout and users
3. [Environment](/environment): the shared `.env` file explained
4. Optional UI: [Dockge](/dockge) or [Portainer](/portainer)
5. Optional VPN for downloads: [Gluetun](/gluetun)

## Catalog

### Media

Media servers, requests and companions.

| Stack | What it does | Main image |
|---|---|---|
| [Dedupe](/dedupe) | Find duplicate media across Jellyfin, Emby and Plex | `spaceinvaderone/jellemplex-dedupe:latest` |
| [Dispatcharr](/dispatcharr) | IPTV / M3U stream and EPG manager | `ghcr.io/dispatcharr/dispatcharr:latest` |
| [Jellyfin](/jellyfin) | Free, open-source media server | `jellyfin/jellyfin:latest` |
| [Jellyseerr](/jellyseerr) | Media request manager (deprecated, use Seerr) | `fallenbagel/jellyseerr:latest` |
| [MeTube](/metube) | Web UI for yt-dlp video downloads | `ghcr.io/alexta69/metube` |
| [Ombi](/ombi) | Media request portal (deprecated, use Seerr) | `lscr.io/linuxserver/ombi:latest` |
| [Plex](/plex) | Plex Media Server with hardware transcoding | `ghcr.io/hotio/plex` |
| [Seerr](/seerr) | Media request manager (successor to Jellyseerr/Overseerr) | `ghcr.io/seerr-team/seerr:latest` |
| [Tautulli](/tautulli) | Plex monitoring and statistics | `lscr.io/linuxserver/tautulli:latest` |
| [Wizarr](/Wizarr) | User invitation portal for Plex and Jellyfin (also part of Arr Stack) | `ghcr.io/wizarrrr/wizarr:latest` |

### Downloads

VPN-protected download clients and the *arr automation suite.

| Stack | What it does | Main image |
|---|---|---|
| [Arr Download](/arr-download) | qBittorrent and SABnzbd behind a Gluetun VPN | `lscr.io/linuxserver/qbittorrent:latest` |
| [Arr Download 1](/arr-download-1) | Second VPN-routed qBittorrent instance | `lscr.io/linuxserver/qbittorrent:latest` |
| [Arr Stack](/arr-stack) | Sonarr, Radarr, Lidarr, Prowlarr, Bazarr and more behind Gluetun (includes Wizarr) | `lscr.io/linuxserver/sonarr:latest` |

### Photos and cameras

Photo libraries and video surveillance.

| Stack | What it does | Main image |
|---|---|---|
| [Frigate NVR](/frigate) | Camera recording with AI object detection | `ghcr.io/blakeblackshear/frigate:0.18.0` |
| [Immich](/immich) | Photo and video library (Google Photos alternative) | `ghcr.io/immich-app/immich-server:release` |

### Documents

Files, sharing, conversion, diagrams and e-signatures.

| Stack | What it does | Main image |
|---|---|---|
| [ConvertX](/convertx) | Web file converter for hundreds of formats | `ghcr.io/c4illin/convertx:latest` |
| [Directory Lister](/directorylister) | Browsable web index of a shared folder | `directorylister/directorylister:latest` |
| [DocuSeal](/docuseal) | Document signing (DocuSign alternative) | `docuseal/docuseal:latest` |
| [Double Commander](/doublecommander) | Dual-pane file manager (deprecated, stack removed) | `lscr.io/linuxserver/doublecommander:latest` |
| [diagrams.net (draw.io)](/drawio) | Diagram and network map editor | `jgraph/drawio` |
| [File Browser](/filebrowser) | Simple web file manager | `filebrowser/filebrowser:s6` |
| [Filestash](/filestash) | Web file manager with Collabora office editing | `machines/filestash:latest` |

### Identity and security

Single sign-on, passwords and remote access.

| Stack | What it does | Main image |
|---|---|---|
| [Apache Guacamole](/guacamole) | Clientless RDP, SSH and VNC gateway | `guacamole/guacd:latest` |
| [Keycloak](/keycloak) | Identity provider and single sign-on (OIDC/SAML) | `quay.io/keycloak/keycloak:26.4.7` |
| [oauth2-proxy](/oauth2-proxy) | SSO auth gateway (Caddy forward_auth + Keycloak) | `quay.io/oauth2-proxy/oauth2-proxy:v7.12.1` |
| [RustDesk Server](/Rustdesk) | Self-hosted remote desktop relay (hbbs + hbbr) | `rustdesk/rustdesk-server:latest` |
| [UniFi Network Application](/unifi) | Self-hosted UniFi Network controller | `lscr.io/linuxserver/unifi-network-application:latest` |
| [Vaultwarden](/vaultwarden) | Bitwarden-compatible password manager | `vaultwarden/server:latest` |

### Monitoring

Metrics, logs, dashboards and GPU/network monitoring.

| Stack | What it does | Main image |
|---|---|---|
| [Grafana Alloy](/alloy) | Telemetry collector that ships per-container metrics to Prometheus | `grafana/alloy:latest` |
| [DCGM Exporter](/dcgm-exporter) | NVIDIA GPU metrics for Prometheus | `nvcr.io/nvidia/k8s/dcgm-exporter:2.0.13-2.1.2-ubuntu18.04` |
| [Dozzle](/dozzle) | Real-time Docker container log viewer | `amir20/dozzle:latest` |
| [Grafana](/grafana) | Dashboards for Prometheus and other data sources | `grafana/grafana:latest` |
| [Graphite Exporter](/graphite-exporter) | Accepts Graphite metrics and exposes them to Prometheus | `prom/graphite-exporter:latest` |
| [Netdata](/netdata) | Real-time host and container monitoring | `netdata/netdata:latest` |
| [nvtop](/nvtop) | NVIDIA GPU process monitor | `69guitar1015/nvtop:latest` |
| [Prometheus](/prometheus) | Metrics database and scraper | `prom/prometheus:latest` |
| [Speedtest Tracker](/speedtest) | Scheduled internet speed tests with history | `lscr.io/linuxserver/speedtest-tracker:latest` |
| [WatchYourLAN](/watchyourlan) | LAN device discovery and inventory | `ghcr.io/aceberg/watchyourlan:latest` |

### Developer and IT tools

Editors, database UIs and handy utilities.

| Stack | What it does | Main image |
|---|---|---|
| [Adminer](/adminer) | Lightweight web UI for MySQL, PostgreSQL and other databases | `adminer:latest` |
| [Bambu Studio](/bambustudio) | Bambu Lab 3D-printer slicer in the browser | `lscr.io/linuxserver/bambustudio:latest` |
| [code-server](/code-server) | VS Code in the browser | `lscr.io/linuxserver/code-server:latest` |
| [CyberChef](/cyberchef) | Encode, decode and transform data in the browser | `ghcr.io/tamcore/cyberchef:latest` |
| [DbGate](/dbgate) | Web database client and manager | `dbgate/dbgate:latest` |
| [IT-Tools](/it-tools) | Collection of handy developer and IT utilities | `corentinth/it-tools:latest` |
| [Rackula](/rackula) | Rack and datacenter layout planner | `ghcr.io/rackulalives/rackula:latest` |

### Stack management

Managing, updating and wiring up your stacks.

| Stack | What it does | Main image |
|---|---|---|
| [Backrest](/backrest) | Web UI for Restic backups | `garethgeorge/backrest:latest` |
| [Dockge](/dockge) | Web UI for managing Docker Compose stacks | `louislam/dockge:1` |
| [Homarr](/homarr) | Homelab dashboard for app shortcuts and widgets | `ghcr.io/homarr-labs/homarr:latest` |
| [Mailrise](/mailrise) | SMTP gateway that turns emails into push notifications | `yoryan/mailrise:latest` |
| [Nebula Sync](/nebula) | Syncs Pi-hole configuration between instances | `ghcr.io/lovelaze/nebula-sync:latest` |
| [Watchtower](/watchtower) | Automatic container image updates | `containrrr/watchtower:latest` |

### Productivity and reading

Reading and news.

| Stack | What it does | Main image |
|---|---|---|
| [13ft](/13ft) | Self-hosted paywall and ad bypass proxy for reading articles | `ghcr.io/wasi-master/13ft:latest` |
| [FreshRSS](/freshrss) | RSS and news feed reader | `freshrss/freshrss:latest` |

### Gaming

Game-related helpers.

| Stack | What it does | Main image |
|---|---|---|
| [ArchiSteamFarm](/archisteamfarm) | Steam card farming and bot manager | `justarchi/archisteamfarm:released` |

## All stacks (A-Z)

| Stack folder | Guide | Category |
|---|---|---|
| `13ft` | [13ft](/13ft) | Productivity and reading |
| `adminer` | [Adminer](/adminer) | Developer and IT tools |
| `alloy` | [Grafana Alloy](/alloy) | Monitoring |
| `archisteamfarm` | [ArchiSteamFarm](/archisteamfarm) | Gaming |
| `arr_download` | [Arr Download](/arr-download) | Downloads |
| `arr_download_1` | [Arr Download 1](/arr-download-1) | Downloads |
| `arr_stack` | [Arr Stack](/arr-stack) | Downloads |
| `backrest` | [Backrest](/backrest) | Stack management |
| `bambustudio` | [Bambu Studio](/bambustudio) | Developer and IT tools |
| `code-server` | [code-server](/code-server) | Developer and IT tools |
| `convertx` | [ConvertX](/convertx) | Documents |
| `cyberchef` | [CyberChef](/cyberchef) | Developer and IT tools |
| `dbgate` | [DbGate](/dbgate) | Developer and IT tools |
| `dcgm-exporter` | [DCGM Exporter](/dcgm-exporter) | Monitoring |
| `dedupe` | [Dedupe](/dedupe) | Media |
| `directorylister` | [Directory Lister](/directorylister) | Documents |
| `dispatcharr` | [Dispatcharr](/dispatcharr) | Media |
| `dockge` | [Dockge](/dockge) | Stack management |
| `docuseal` | [DocuSeal](/docuseal) | Documents |
| `doublecommander` | [Double Commander](/doublecommander) (deprecated) | Documents |
| `dozzle` | [Dozzle](/dozzle) | Monitoring |
| `drawio` | [diagrams.net (draw.io)](/drawio) | Documents |
| `filebrowser` | [File Browser](/filebrowser) | Documents |
| `filestash` | [Filestash](/filestash) | Documents |
| `freshrss` | [FreshRSS](/freshrss) | Productivity and reading |
| `frigate` | [Frigate NVR](/frigate) | Photos and cameras |
| `grafana` | [Grafana](/grafana) | Monitoring |
| `graphite-exporter` | [Graphite Exporter](/graphite-exporter) | Monitoring |
| `guacamole` | [Apache Guacamole](/guacamole) | Identity and security |
| `homarr` | [Homarr](/homarr) | Stack management |
| `immich` | [Immich](/immich) | Photos and cameras |
| `it-tools` | [IT-Tools](/it-tools) | Developer and IT tools |
| `jellyfin` | [Jellyfin](/jellyfin) | Media |
| `jellyseerr` | [Jellyseerr](/jellyseerr) | Media |
| `keycloak` | [Keycloak](/keycloak) | Identity and security |
| `mailrise` | [Mailrise](/mailrise) | Stack management |
| `metube` | [MeTube](/metube) | Media |
| `nebula` | [Nebula Sync](/nebula) | Stack management |
| `netdata` | [Netdata](/netdata) | Monitoring |
| `nvtop` | [nvtop](/nvtop) | Monitoring |
| `oauth2-proxy` | [oauth2-proxy](/oauth2-proxy) | Identity and security |
| `ombi` | [Ombi](/ombi) (deprecated, use Seerr) | Media |
| `plex` | [Plex](/plex) | Media |
| `prometheus` | [Prometheus](/prometheus) | Monitoring |
| `rackula` | [Rackula](/rackula) | Developer and IT tools |
| `rustdesk` | [RustDesk Server](/Rustdesk) | Identity and security |
| `seerr` | [Seerr](/seerr) | Media |
| `speedtest` | [Speedtest Tracker](/speedtest) | Monitoring |
| `tautulli` | [Tautulli](/tautulli) | Media |
| `unifi` | [UniFi Network Application](/unifi) | Identity and security |
| `vaultwarden` | [Vaultwarden](/vaultwarden) | Identity and security |
| `watchtower` | [Watchtower](/watchtower) | Stack management |
| `watchyourlan` | [WatchYourLAN](/watchyourlan) | Monitoring |
| `wizarr` | [Wizarr](/Wizarr) | Media |

## Other guides
- [Deluge](/deluge), [Transmission](/transmission), [SABnzbd](/sabnzdb) and [Tdarr](/tdarr): older single-app examples
- [Home Assistant](/home-assistant)
- [Useful Commands](/usefulcommands)

## Security note
Never publish your `.secret` file, live passwords, API keys, claim tokens or database URLs. Put services that have no login of their own behind a reverse proxy with authentication before exposing them to the internet.

---
*Questions? Ask on the [forums](https://forums.thehomestack.io/).*
