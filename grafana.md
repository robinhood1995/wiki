---
title: Grafana
description: Grafana dashboards for Prometheus and other data sources, with plugins and a memory limit, using Docker Compose.
published: true
date: 2026-10-20T20:01:23.503Z
tags: docker, docker-compose, grafana, homelab, monitoring, prometheus, dashboards
editor: markdown
dateCreated: 2026-09-08T21:11:08.018Z
---

# Grafana

> Grafana dashboards for Prometheus and other data sources, with plugins and a memory limit, using Docker Compose.
{.is-info}

_Last updated: Oct 2026_

## Overview

Grafana turns metrics and logs into dashboards and alerts. Connect it to Prometheus, Loki, InfluxDB, MySQL and many other sources, then build your own panels or import community dashboards for Linux hosts, Docker, pfSense, Synology and GPUs.

**Why you'd want it:** One screen that shows the health of every server, container and network device in your lab.

| Item | Value |
|---|---|
| Stack folder | `/opt/stacks/grafana` |
| Category | Monitoring |
| Main image | `grafana/grafana:latest` |
| Services | 1 |
| Web UI | http://<SERVER_IP>:30000 |
| Official docs | https://grafana.com/docs/grafana/latest/setup-grafana/installation/docker/ |

## Requirements

- A Linux host with Docker Engine and the Docker Compose plugin (see [Docker](/docker))
- The shared folder layout and `.env`/`.secret` files from [Docker Folders](/dockerfolders) and [Environment](/environment)
- A data source such as [Prometheus](/prometheus)

## Compose

Folder layout:

```text
/opt/stacks/
├── _shared/
│   ├── .env        # shared, non-secret settings
│   └── .secret     # passwords, tokens and keys (chmod 600, never commit)
└── grafana/
    └── compose.yaml
/opt/appdata/grafana/   # persistent application data
```

`/opt/stacks/grafana/compose.yaml`:

```yaml
---
services:
  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    hostname: grafana
    env_file:
      - /opt/stacks/_shared/.env
      - /opt/stacks/_shared/.secret
    mem_limit: 512m
    cpu_shares: 512
    security_opt:
      - no-new-privileges:true
    user: 0:0
    healthcheck:
      test: wget --no-verbose --tries=1 --spider http://localhost:3000/api/health
    environment:
      - GF_INSTALL_PLUGINS=grafana-clock-panel,grafana-simple-json-datasource,natel-discrete-panel,grafana-piechart-panel
    logging:
      driver: json-file
      options:
        max-file: ${DOCKERLOGGING_MAXFILE:-10}  # Max number of log files
        max-size: ${DOCKERLOGGING_MAXSIZE:-200k}  # Max file size
    ports:
      - 30000:3000
    volumes:
      - /opt/appdata/grafana:/var/lib/grafana:rw
    restart: unless-stopped
```

## Environment variables

Compose reads these from the shared files with `--env-file`. Replace every `<PLACEHOLDER>` with your own value.

`/opt/stacks/_shared/.env`:

```ini
PUID=1000
PGID=1000
TZ=<TZ>
DOCKERLOGGING_MAXFILE=10
DOCKERLOGGING_MAXSIZE=200k
```

| Variable | File | Purpose | Example |
|---|---|---|---|
| `PUID` | `.env` | User ID the container runs as (see Docker Folders) | `1000` |
| `PGID` | `.env` | Group ID the container runs as | `1000` |
| `TZ` | `.env` | Time zone in tz database format, e.g. America/New_York | `<TZ>` |
| `DOCKERLOGGING_MAXFILE` | `.env` | Number of rotated container log files to keep | `10` |
| `DOCKERLOGGING_MAXSIZE` | `.env` | Maximum size of each container log file | `200k` |

## Ports

| Host port | Container port | Protocol | Service | Purpose |
|---|---|---|---|---|
| 30000 | 3000 | tcp | `grafana` | Web UI |

## Volumes

| Host path | Container path | Service | Purpose |
|---|---|---|---|
| `/opt/appdata/grafana` | `/var/lib/grafana` | `grafana` | Dashboards, users, plugins and SQLite database |

## Resource limits

Recommended limits, already set in the compose file under `deploy.resources`. Raise them if the app is starved, or lower them on small hosts.

| Service | CPU limit | Memory limit | Memory reservation | GPU |
|---|---|---|---|---|
| `grafana` | - | 512m | - | - |

## Setup

1. Create the folders:
   ```bash
   sudo mkdir -p /opt/stacks/grafana /opt/appdata/grafana
   sudo chown -R $USER: /opt/stacks/grafana
   ```
2. Add the variables above to `/opt/stacks/_shared/.env` and `/opt/stacks/_shared/.secret` (`chmod 600 .secret`).
3. Save the compose file as `/opt/stacks/grafana/compose.yaml`.
4. Check the configuration, then start the stack:
   ```bash
   cd /opt/stacks/grafana
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret config --quiet
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
   ```
5. Watch the first start: `docker compose logs -f`.
6. Open `http://<SERVER_IP>:30000` and sign in as `admin` / `admin`; Grafana asks you to set a new password.
7. Add a data source under **Connections > Data sources** (Prometheus URL `http://<SERVER_IP>:9090`).
8. Import dashboards from grafana.com: 1860 (Node Exporter Full), 193 (Docker), 12239 (NVIDIA DCGM).

> Tip: create a small wrapper such as `alias dc='docker compose --env-file /opt/stacks/_shared/.env --env-file /opt/stacks/_shared/.secret'` so every stack uses the same env files.
{.is-success}

## Updating

```bash
cd /opt/stacks/grafana
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret pull
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
docker image prune -f
```

Back up `/opt/appdata/grafana` before major upgrades. [Watchtower](/watchtower) can automate updates for stateless apps.

## Troubleshooting

- Check status and logs: `docker compose ps` and `docker compose logs --tail=100`.
- **Permission denied on /var/lib/grafana:** this example runs as `user: 0:0` to avoid ownership problems. To run unprivileged, use `user: "472"` and `chown -R 472:0 /opt/appdata/grafana`.
- **Plugin install fails on start:** remove deprecated plugins from `GF_INSTALL_PLUGINS` (the pie chart panel is built in since Grafana 8).
- **Out of memory with many dashboards:** raise `mem_limit`.
- **A variable is empty or reported missing (`:?err`):** run the `config` command above to see which variable is not set in your env files.

## Related

- [Docker Stack Catalog](/docker-stacks)
- [Prometheus](/prometheus)
- [Grafana Alloy](/alloy)
- [DCGM Exporter](/dcgm-exporter)
- [Graphite Exporter](/graphite-exporter)
- [Netdata](/netdata)

---
*Educational example. Review every path, port and credential before you deploy, and never publish your `.secret` file.*
