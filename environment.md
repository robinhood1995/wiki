---
title: Environment
description: 
published: true
date: 2025-10-19T21:31:41.420Z
tags: 
editor: markdown
dateCreated: 2025-10-19T19:23:39.587Z
---

# <img src="/images/windows-terminal.png" width="50" style="vertical-align:middle;margin-right:4px">What's a Docker .ENV file?
A Docker .env file is a plain-text file used to define environment variables for Docker containers, particularly when working with Docker Compose. It serves as a central location to store configuration settings, including sensitive information like database credentials or API keys, separate from your application code or Dockerfile.

## Breakdown and key aspects:
### Purpose:
It allows you to externalize configuration, making it easier to manage and share settings across different environments (development, staging, production) without modifying the core application or Docker image.
### Content:
It contains key-value pairs, where each line defines an environment variable in the format VARIABLE_NAME=value.
### Usage with Docker Compose:
When using Docker Compose, the .env file (typically placed in the same directory as your docker-compose.yaml file) is automatically loaded and its variables are made available for interpolation within the docker-compose.yaml file and to the services defined within it. You can also explicitly specify an environment file using the env_file directive in your docker-compose.yaml.
### Usage with docker run:
You can also use an .env file with the docker run command by using the --env-file flag to pass the variables to a single container.
### Precedence:
It's important to note that environment variables set directly on the host system or within the docker-compose.yaml file can override values defined in the .env file.

## Example:
These are examples that we use throughout this site on out projects.
```ini
PUID=1024
PGID=100
TZ=America/Montreal
UMASK=002
DOCKERLOGGING_MAXFILE=10
DOCKERLOGGING_MAXSIZE=200k
DOCKERFOLDER=/volume1/docker
TORRENTDOWNLOAD=/mnt/data/torrents
USENETDOWNLOAD=/mnt/data/usenet
SERVAAR=/mnt/data
MEDIAFOLDER=/mnt/data/media
VPN_TYPE=openvpn
SERVER_COUNTRIES=Albania,Algeria,Angola,Argentina,Australia,Austria,Azerbaijan
OPENVPN_USERNAME=
OPENVPN_PASSWORD=
TP_THEME=organizr
RADARR_API_KEY=
SONARR_API_KEY=
LIDARR_API_KEY=
MYLAR_API_KEY=
PROWLARR_API_KEY=
LAZYLIBRARIAN_API_KEY=
WHISPARR_API_KEY=
READARR_API_KEY=
PLEX_API_KEY=
```