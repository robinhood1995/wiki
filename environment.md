---
title: Environment
description: Shared .env and .secret files used by every stack guide, with /opt/stacks paths and placeholders.
published: true
date: 2026-11-22T06:33:32.245Z
tags: 
editor: markdown
dateCreated: 2025-10-19T19:23:39.587Z
---

# Environment

> Shared `.env` and `.secret` files used by every stack guide on this wiki.
{.is-info}

_Last updated: Nov 2026_

## What's a Docker `.env` file?

A Docker `.env` file is a plain-text file of environment variables for containers, especially when you use Docker Compose. It keeps configuration — and, in a separate secrets file, credentials — out of your compose file and images.

### Purpose

Externalizing configuration makes it easier to reuse the same compose file on different machines without editing the file itself.

### Content

Each line is a `VARIABLE_NAME=value` pair. Quotes are only needed when a value contains spaces.

### Usage with Docker Compose

The stack guides on this wiki do **not** rely on a compose-directory `.env` alone. They load two shared files explicitly:

```bash
docker compose --env-file /opt/stacks/_shared/.env --env-file /opt/stacks/_shared/.secret up -d
```

You can also pass a file to `docker run` with `--env-file`.

### Precedence

Variables set in the shell or in the compose `environment:` block override values from the env files. Later `--env-file` arguments override earlier ones, so `.secret` wins over `.env` for the same key.

## Files used on this wiki

```text
/opt/stacks/_shared/.env      # shared, non-secret settings
/opt/stacks/_shared/.secret   # passwords, tokens and keys (chmod 600, never commit)
```

See [Docker Folders](/dockerfolders) for the rest of the directory layout (`/opt/stacks`, `/opt/appdata`, `/srv/media`, `/srv/data`).

### Example `.env`

Replace every `<PLACEHOLDER>` with your own value. `PUID` and `PGID` should match a user on your host; `1000` is a common default on desktop Linux, not a requirement.

```ini
PUID=1000
PGID=1000
TZ=<TZ>
DOCKERLOGGING_MAXFILE=10
DOCKERLOGGING_MAXSIZE=200k
```

Optional paths used by some download and *arr* guides:

```ini
MEDIAFOLDER=/srv/media
DOWNLOADFOLDER=/srv/data
```

### Example `.secret`

Keep this file mode `600` and never publish it. API keys, VPN credentials and database passwords belong here. The empty keys below are placeholders only:

```ini
OPENVPN_USERNAME=
OPENVPN_PASSWORD=
RADARR_API_KEY=
SONARR_API_KEY=
LIDARR_API_KEY=
PROWLARR_API_KEY=
PLEX_API_KEY=
```

Fill only the variables your stacks actually use.

## Related

- [Docker Folders](/dockerfolders)
- [Docker Stack Catalog](/docker-stacks)
- [Docker](/docker)

---
*Educational example. Review every path, port and credential before you deploy, and never publish your `.secret` file.*
