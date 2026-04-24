---
title: Rustdesk
description: 
published: true
date: 2026-04-24T20:01:29.984Z
tags: 
editor: markdown
dateCreated: 2026-04-24T19:43:25.326Z
---

# <img src="/images/rustdesk.png" width="50" style="vertical-align:middle;margin-right:4px" class="tab-icon"> What is Rustdesk?
RustDesk is a free, open-source remote desktop software application that allows users to securely access and control computers and devices remotely, serving as an alternative to TeamViewer or AnyDesk. It features self-hosting capabilities to ensure data privacy and supports multiple platforms, including Windows, macOS, Linux, iOS, and Android. 

# Deploy Rustdesk
# {.tabset}
## <img src="/images/docker.png" width="25" class="tab-icon"> Docker
```yaml
services:
  hbbs: # ID / signaling server
    container_name: hbbs
    image: rustdesk/rustdesk-server:latest
    command: hbbs
    logging:
      driver: json-file
      options:
        max-file: ${DOCKERLOGGING_MAXFILE:-10}
        max-size: ${DOCKERLOGGING_MAXSIZE:-200k}
    ports:
      - "21115:21115"
      - "21116:21116"
      - "21116:21116/udp"
      - "21118:21118"
    volumes:
      - ${DOCKERFOLDER}/rustdesk/data:/root
    depends_on:
      - hbbr
    netowrk_mode: host
    restart: unless-stopped
  hbbr: # Relay server
    container_name: hbbr
    image: rustdesk/rustdesk-server:latest
    command: hbbr
    logging:
      driver: json-file
      options:
        max-file: ${DOCKERLOGGING_MAXFILE:-10}
        max-size: ${DOCKERLOGGING_MAXSIZE:-200k}
    ports:
      - "21117:21117"
      - "21119:21119"
    volumes:
      - ${DOCKERFOLDER}/rustdesk/data:/root
    netowrk_mode: host
    restart: unless-stopped
```
## <img src="/images/docker.png" width="25" class="tab-icon"> pfSense Ports
Table of Ports (using HAProxy)
| Port | Protocol | Purpose | Recommended pfSense Method |
| :--- | :--- | :--- | :--- |
| **21115** | TCP | ID Server (hbbs) | HAProxy (TCP Mode) |
| **21116** | TCP | TCP Hole Punching | HAProxy (TCP Mode) |
| **21116** | UDP | Heartbeat/Status | NAT Port Forward |
| **21117** | TCP | Relay Server (hbbr) | HAProxy (TCP Mode) |
| **21118/9** | TCP | WebSockets (Web UI) | HAProxy (TCP/SSL Mode) |

### Environment File
📌 .ENV / .env files code needed for setting the above containers have a look here for all variables
[Environment File](/environment)

# Rustdesk Configuration
> 
> Read the [official documentation](https://rustdesk.com/)
{.is-success}
