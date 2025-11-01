---
title: Transmission
description: 
published: true
date: 2025-10-29T22:36:07.549Z
tags: 
editor: markdown
dateCreated: 2025-10-19T22:40:36.863Z
---

# <img src="/images/transmission.png" width="50" style="vertical-align:middle;margin-right:4px">What's Transmission?

Transmission is a lightweight, free, and open-source BitTorrent client that simplifies downloading and sharing files over the internet. It is known for its simple, user-friendly interface and efficient performance, making it a popular choice for macOS, Windows, and Linux users. It supports features like encrypted connections, remote control via a web interface, and automatic port mapping

# Deploy Transmission
# {.tabset}
## <img src="/images/docker.png" width="25" class="tab-icon"> Built-in OpenVPN

In this Docker compose file we are using the <img src="/images/proton-vpn.png" width="25" class="tab-icon"> ProtonVPN services.
```yaml
services:
###########################################################################
##  Docker Compose File:  Transmission (haugene)
##  Function:             Torrent Download Client
##  Documentation:        https://github.com/haugene/docker-transmission-openvpn
###########################################################################
  transmission:
    image: haugene/transmission-openvpn
    container_name: transmission-openvpn
    cap_add:
        - NET_ADMIN # Required for VPN functionality if using haugene/transmission-openvpn
    environment:
        - CREATE_TUN_DEVICE=true
        - OPENVPN_PROVIDER=custom
        - OPENVPN_CONFIG=hr-15.protonvpn.udp # Replace with your .ovpn filename
        - OPENVPN_USERNAME=${OPENVPN_USERNAME:?err}
        - OPENVPN_PASSWORD=${OPENVPN_PASSWORD:?err}
        - OPENVPN_OPTS=--inactive 3600 --ping 10 --ping-exit 60
        - WEBPROXY_ENABLED=false
        - LOCAL_NETWORK=${LAN_NETWORK:?err}  # Your local network range
        - PUID=${PUID:?err}    # User ID
        - PGID=${PGID:?err}    # Group ID
        - TZ=${TZ:?err}        # TimeZone
        - UMASK=${UMASK:?err}  # File Umask
        - DEBUG=true
        - TRANSMISSION_SCRAPE_PAUSED_TORRENTS_ENABLED=false
        - TRANSMISSION_UMASK=${UMASK:?err}
    volumes:
        - ${DOCKERFOLDER:?err}/transmission-openvpn/ovpn_files/:/etc/openvpn/custom
        - ${DOCKERFOLDER:?err}/transmission-openvpn/config/:/config
        - ${MEDIAFOLDER}:/data
        - ${MEDIAFOLDER}/torrents:/downloads # Torrent downloads (complete and incomplete)
    logging:
      driver: json-file
      options:
        max-file: ${DOCKERLOGGING_MAXFILE:?err}  # Max number of log files
        max-size: ${DOCKERLOGGING_MAXSIZE:?err}  # Max file size
    ports:
        - 9091:9091 # Transmission Web UI
        - 51413:51413 # Transmission Peer Port (adjust if using port forwarding)
        - 51413:51413/udp # Transmission Peer Port (adjust if using port forwarding)
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
      - 8085:8085/tcp   # WebUI Portal: qBittorrent
      - 8888:8888/tcp   # Gluetun Local Network HTTP proxy
      - 8388:8388/tcp   # Gluetun Local Network Shadowsocks
      - 8388:8388/udp   # Gluetun Local Network Shadowsocks
      - 6881:6881/tcp   # Transmission Torrent Port TCP
      - 6881:6881/udp   # Transmission Torrent Port UDP
    restart: unless-stopped

###########################################################################
##  Docker Compose File:  Transmission
##  Function:             Torrent Download Client
##  Documentation:        https://
###########################################################################
  transmission:
    image: lscr.io/linuxserver/transmission:latest
    container_name: transmission
    network_mode: "service:gluetun"
    depends_on:
      gluetun:
        condition: service_healthy
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
        - ${DOCKERFOLDER:?err}/transmission/config/:/config
        - ${MEDIAFOLDER:?err}:/data/media
        - ${TORRENTDOWNLOAD:?err}:/data/torrents # Torrent downloads (complete and incomplete)
        #- /path/to/your/watch:/watch # Optional: watch directory for auto-adding torrents
    restart: unless-stopped
```
### Environment File
📌 .ENV / .env files code needed for setting the above containers have a look here for all variables
[Environment File](/environment)


