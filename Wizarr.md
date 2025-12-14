---
title: Wizarr
description: 
published: true
date: 2025-12-14T06:33:07.793Z
tags: 
editor: markdown
dateCreated: 2025-12-14T04:51:54.874Z
---

# <img src="/images/wizarr.png" width="50" style="vertical-align:middle;margin-right:4px" class="tab-icon"> What is Wizarr?
Wizarr is an automated user invitation system compatible with Plex, Jellyfin and Emby. You can create a unique link, share it with a user, and they will be invited to your Media Server after they complete the simple signup process!

# <img src="/images/docker.png" width="50" style="vertical-align:middle;margin-right:4px" class="tab-icon"> 1 · Deploy Wizarr
```yaml
services:
  wizarr:
    container_name: wizarr
    restart: unless-stopped
    image: ghcr.io/wizarrrr/wizarr:latest
    ports:
      - 5690:5690
    environment:
      - APP_URL=https://example.domain.com
      - TZ=America/Montreal
    volumes:
      - /mnt/tank/configs/wizarr:/data/database
      - /mnt/tank/configs/wizarr/wizard:/data/wizard_steps
```
1. Change the `APP_URL` to match your domain
1. The config directory assumes you follow the standard Folder Structure. If you do not it needs to be modified to the correct volume.

> Use **Generic** for the dataset preset in TrueNAS
{.is-info}


# 2 · Wizarr Configuration
> 
> Read the [official documentation](https://docs.wizarr.dev/)
{.is-success}
1. Create an admin account
1. Navigate to **Settings Servers**
1. Add your media server
> You will first need to create an API Key for Jellyfin by navigating to the **Administration Dashboard** → **API Keys** and clicking the `+` in the top left
{.is-info}
