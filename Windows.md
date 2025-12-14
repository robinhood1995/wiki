---
title: Windows
description: 
published: true
date: 2025-12-14T06:27:28.283Z
tags: 
editor: markdown
dateCreated: 2025-12-14T06:27:28.283Z
---

# <img src="/images/microsoft-windows.png" width="50" style="vertical-align:middle;margin-right:4px" class="tab-icon">What is the Windows Docker Container?
The Windows 11 Operating System within a docker container.

# <img src="/images/docker.png" width="50" style="vertical-align:middle;margin-right:4px" class="tab-icon"> 1 · Deploy Windows
```yaml
services:
  windows:
    image: dockurr/windows
    container_name: windows11
    environment:
      VERSION: "11"
      USERNAME: "admin"
      PASSWORD: "admin"
      RAM_SIZE: "16G"
      CPU_CORES: "4"
#      VM_NET_DEV: "NAME"
      REGION: "en-US"
      KEYBOARD: "en-US"
    devices:
      - /dev/kvm
      - /dev/net/tun
    cap_add:
      - NET_ADMIN
    volumes:
      - ${DOCKERFOLDER:?err}/windows/windows11:/storage
    ports:
      - 8009:8006       # Web Console
      - 3389:3389/tcp   # RDP Port
      - 3389:3389/udp   # RDP Port
    stop_grace_period: 2m
    restart: always
```