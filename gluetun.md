---
title: Gluetun
description: VPN client sidecar compose example for download stacks.
published: true
date: 2026-09-08T21:30:07.819Z
tags: 
editor: markdown
dateCreated: 2026-09-08T21:11:08.018Z
---

# Gluetun

VPN client sidecar compose example for download stacks.

## Disclaimer

Educational example only. Review every volume path, port, and credential before you run this. Do **not** copy production secrets into your wiki or git repo.

## Prerequisites

- Docker + Docker Compose (or Portainer / Dockge)
- A shared env file pattern (optional): `_shared/.env` for non-secrets, `_shared/.secret` for credentials
- Storage paths adjusted for your host

## Quick start

```bash
mkdir -p /path/to/docker/stacks/gluetun
cd /path/to/docker/stacks/gluetun
# paste compose.yaml below, then:
docker compose pull
docker compose up -d
```

## compose.yaml

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
    env_file:
      - /path/to/docker/stacks/_shared/.env
      - /path/to/docker/stacks/_shared/.secret
    environment:
      - TZ=${TZ:-UTC}
      - UPDATER_PERIOD=24h
      - VPN_SERVICE_PROVIDER=CHANGE_ME
      - VPN_TYPE=wireguard
      - BLOCK_MALICIOUS=off
      # Put provider credentials in _shared/.secret, for example:
      # OPENVPN_USER / OPENVPN_PASSWORD or WIREGUARD_PRIVATE_KEY
      - SERVER_COUNTRIES=CHANGE_ME
      - VPN_PORT_FORWARDING=on
      - SHADOWSOCKS=off
    volumes:
      - /path/to/docker/data/gluetun:/gluetun
    logging:
      driver: json-file
      options:
        max-file: ${DOCKERLOGGING_MAXFILE:-10}
        max-size: ${DOCKERLOGGING_MAXSIZE:-200k}
    ports:
      - 8888:8888/tcp   # HTTP proxy (optional)
      - 8388:8388/tcp   # Shadowsocks (optional)
      - 8388:8388/udp
    restart: unless-stopped
```

## Notes

- Provider credentials belong in `_shared/.secret`.
- Route dependent containers with `network_mode: "service:gluetun"` when you want VPN egress.
- See also the older Deluge + Gluetun example on [Deluge](/deluge).

## Related

- [Docker Homelab Stacks](/docker-stacks)
- [Dockge](/dockge)
- [Portainer](/portainer)

