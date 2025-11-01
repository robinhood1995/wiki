---
title: SABnzdb
description: 
published: true
date: 2025-10-29T22:35:27.539Z
tags: 
editor: markdown
dateCreated: 2025-10-29T22:34:31.982Z
---

# <img src="/images/sabnzbd.png" width="50" style="vertical-align:middle;margin-right:4px">What's SABnzdb?

SABnzbd is a Usenet download client, not a torrent client, but it functions similarly in that it downloads files from a network. The confusion may arise because both SABnzbd and torrents use a file to initiate a download, but the underlying network is different. SABnzbd uses .nzb files to download data from Usenet servers, whereas torrents use .torrent files to download from a peer-to-peer network.

# Deploy SABnzdb
# {.tabset}
## <img src="/images/docker.png" width="25" class="tab-icon"> Compose

In this Docker compose file we are using the <img src="/images/proton-vpn.png" width="25" class="tab-icon"> ProtonVPN services.
```yaml
services:
###########################################################################
##  Docker Compose File:  SABnzbd (LinuxServer.io)
##  Function:             Usenet Download Client
##  Documentation:        https://docs.linuxserver.io/images/docker-sabnzbd
###########################################################################
  sabnzbd:
    image: lscr.io/linuxserver/sabnzbd:latest
    container_name: sabnzbd
    hostname: sabnzdb
    environment:
      - PUID=${PUID:?err:?err}    # User ID
      - PGID=${PGID:?err:?err}    # Group ID
      - TZ=${TZ:?err:?err}        # TimeZone
      - UMASK=${UMASK:?err:?err}  # File Umask
      - DOCKER_MODS=ghcr.io/gilbn/theme.park:sabnzbd
      - TP_THEME=${TP_THEME:?err}
    volumes:
      - ${DOCKERFOLDER:?err}/sabnzbd:/config
      - ${USENETDOWNLOAD:?err}:/data/usenet
      - ${MEDIAFOLDER:?err}:/data/media
    logging:
      driver: json-file
      options:
        max-file: ${DOCKERLOGGING_MAXFILE:?err}  # Max number of log files
        max-size: ${DOCKERLOGGING_MAXSIZE:?err}  # Max file size
    restart: unless-stopped
```
### Environment File
📌 .ENV / .env files code needed for setting the above containers have a look here for all variables
[Environment File](/environment)

## <img src="/images/docker.png" width="25" class="tab-icon"> Stack w/VPN

In this Docker compose file we are using the <img src="/images/proton-vpn.png" width="25" class="tab-icon"> ProtonVPN services.
```yaml
services:
###########################################################################
##  Docker Compose File:    Gluetun (qmcgaw)
##  Function:               VPN Client
##  Documentation:          https://github.com/qdm12/gluetun-wiki
###########################################################################
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
      - VPN_PORT_FORWARDING_UP_COMMAND=/bin/sh -c 'wget -O- --retry-connrefused --post-data "json={\"listen_port\":{{PORTS:?err}:?err}:?err}" http://127.0.0.1:8080/api/v2/app/setPreferences 2>&1'
      - SERVER_COUNTRIES=${SERVER_COUNTRIES:?err}
      - SHADOWSOCKS=on
    volumes:
      - ${DOCKERFOLDER:?err}/gluetun/config:/gluetun
    logging:
      driver: json-file
      options:
        max-file: ${DOCKERLOGGING_MAXFILE:?err}  # Max number of log files
        max-size: ${DOCKERLOGGING_MAXSIZE:?err}  # Max file size
    ports:
      - 8084:8080       # SABnzbd
      - 8890:8888/tcp   # Gluetun Local Network HTTP proxy
      - 8390:8388/tcp   # Gluetun Local Network Shadowsocks
      - 8390:8388/udp   # Gluetun Local Network Shadowsocks
    restart: unless-stopped

###########################################################################
##  Docker Compose File:  SABnzbd (LinuxServer.io)
##  Function:             Usenet Download Client
##  Documentation:        https://docs.linuxserver.io/images/docker-sabnzbd
###########################################################################
  sabnzbd:
    image: lscr.io/linuxserver/sabnzbd:latest
    container_name: sabnzbd
    network_mode: "service:gluetun"
    depends_on:
      gluetun:
        condition: service_healthy
    environment:
      - PUID=${PUID:?err:?err}    # User ID
      - PGID=${PGID:?err:?err}    # Group ID
      - TZ=${TZ:?err:?err}        # TimeZone
      - UMASK=${UMASK:?err:?err}  # File Umask
      - DOCKER_MODS=ghcr.io/gilbn/theme.park:sabnzbd
      - TP_THEME=${TP_THEME:?err}
    volumes:
      - ${DOCKERFOLDER:?err}/sabnzbd:/config
      - ${USENETDOWNLOAD:?err}:/data/usenet
      - ${MEDIAFOLDER:?err}:/data/media
    logging:
      driver: json-file
      options:
        max-file: ${DOCKERLOGGING_MAXFILE:?err}  # Max number of log files
        max-size: ${DOCKERLOGGING_MAXSIZE:?err}  # Max file size
    restart: unless-stopped
```
### Environment File
📌 .ENV / .env files code needed for setting the above containers have a look here for all variables
[Environment File](/environment)


