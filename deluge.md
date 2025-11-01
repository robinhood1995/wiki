---
title: Deluge
description: 
published: true
date: 2025-10-29T22:36:44.941Z
tags: 
editor: markdown
dateCreated: 2025-10-29T22:22:49.866Z
---

# <img src="/images/deluge.png" width="50" style="vertical-align:middle;margin-right:4px">What's Deluge?

Deluge is a free, open-source, and lightweight BitTorrent client used for peer-to-peer file sharing. It uses a client-server model and is available on multiple operating systems, including Windows, macOS, and Linux. Its features, like a rich plugin system, are modular, allowing users to add functionality as needed.

# Deploy Deluge w/VPN Stack

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
      - 8112:8112 # Deluge web UI
      - 6882:6881 # Deluge incoming port (TCP)
      - 6882:6881/udp # Deluge incoming port (UDP)
      - 8889:8888/tcp   # Gluetun Local Network HTTP proxy
      - 8389:8388/tcp   # Gluetun Local Network Shadowsocks
      - 8389:8388/udp   # Gluetun Local Network Shadowsocks
    restart: unless-stopped

###########################################################################
##  Docker Compose File:  deluge (LinuxServer.io)
##  Function:             Torrent Download Client
##  Documentation:        https://docs.linuxserver.io/images/docker-deluge
###########################################################################
  deluge:
    image: linuxserver/deluge
    container_name: deluge
    network_mode: "service:gluetun"
    depends_on:
      gluetun:
        condition: service_healthy
    environment:
      - PUID=${PUID:?err:?err}    # User ID
      - PGID=${PGID:?err:?err}    # Group ID
      - TZ=${TZ:?err:?err}        # TimeZone
      - UMASK=${UMASK:?err:?err}  # File Umask
    volumes:
      - ${DOCKERFOLDER:?err}/deluge/config:/config
      - ${TORRENTDOWNLOAD:?err}:/data/torrents
      - ${MEDIAFOLDER:?err}:/data/media
    restart: unless-stopped
```
### Environment File
📌 .ENV / .env files code needed for setting the above containers have a look here for all variables
[Environment File](/environment)


