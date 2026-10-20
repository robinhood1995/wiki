---
title: Netdata
description: Netdata real-time host and container monitoring with Docker Compose.
published: true
date: 2026-10-18T03:28:18.529Z
tags: docker, docker-compose, homelab, metrics, monitoring, alerts
editor: markdown
dateCreated: 2026-09-08T21:15:35.866Z
---

# Netdata

> Netdata real-time host and container monitoring with Docker Compose.
{.is-info}

_Last updated: Oct 2026_

## Overview

Netdata collects thousands of metrics per second with zero configuration: CPU, memory, disks, network, containers, services and more. It ships with sensible alerts and per-second dashboards.

**Why you'd want it:** Instant, detailed insight into what your server is doing right now.

| Item | Value |
|---|---|
| Stack folder | `/opt/stacks/netdata` |
| Category | Monitoring |
| Main image | `netdata/netdata:latest` |
| Services | 1 |
| Web UI | http://<SERVER_IP>:19999 |
| Official docs | https://learn.netdata.cloud/docs/netdata-agent/installation/docker |

## Requirements

- A Linux host with Docker Engine and the Docker Compose plugin (see [Docker](/docker))
- The shared folder layout and `.env`/`.secret` files from [Docker Folders](/dockerfolders) and [Environment](/environment)
- Host networking and `pid: host` for full visibility

## Compose

Folder layout:

```text
/opt/stacks/
├── _shared/
│   ├── .env        # shared, non-secret settings
│   └── .secret     # passwords, tokens and keys (chmod 600, never commit)
└── netdata/
    └── compose.yaml
/opt/appdata/netdata/   # persistent application data
```

`/opt/stacks/netdata/compose.yaml`:

```yaml
services:
  netdata:
    image: netdata/netdata:latest
    container_name: netdata
    hostname: <SERVER_NAME>  # Optional
    env_file:
      - /opt/stacks/_shared/.env
      - /opt/stacks/_shared/.secret
    pid: host
    cap_add:
      - SYS_PTRACE
      - SYS_ADMIN
    security_opt:
      - apparmor:unconfined  # Required on systems with AppArmor; use 'SELinux' equivalent if needed
    volumes:
      - /opt/appdata/netdata/netdataconfig:/etc/netdata
      - /opt/appdata/netdata/netdatalib:/var/lib/netdata
      - /opt/appdata/netdata/netdatacache:/var/cache/netdata
      # Host mounts for full system monitoring
      - /:/host/root:ro,rslave
      - /etc/passwd:/host/etc/passwd:ro
      - /etc/group:/host/etc/group:ro
      - /etc/localtime:/etc/localtime:ro
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /etc/os-release:/host/etc/os-release:ro
      - /var/log:/host/var/log:ro
      # Docker socket for container monitoring (optional but recommended)
      - /var/run/docker.sock:/var/run/docker.sock:ro
    logging:
      driver: json-file
      options:
        max-file: ${DOCKERLOGGING_MAXFILE:-10}  # Max number of log files
        max-size: ${DOCKERLOGGING_MAXSIZE:-200k}  # Max file size
    # Optional: Health check
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:19999/api/v1/info || exit 1"]
      interval: 2m
      timeout: 20s
      retries: 3
    network_mode: host
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 1000M
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
| host network | - | - | `netdata` | 19999/tcp (dashboard) |

## Volumes

| Host path | Container path | Service | Purpose |
|---|---|---|---|
| `/opt/appdata/netdata/netdataconfig` | `/etc/netdata` | `netdata` | Application data |
| `/opt/appdata/netdata/netdatalib` | `/var/lib/netdata` | `netdata` | Application data |
| `/opt/appdata/netdata/netdatacache` | `/var/cache/netdata` | `netdata` | Application data |
| `/` | `/host/root` | `netdata` | Read-only host view for monitoring (read-only) |
| `/etc/passwd` | `/host/etc/passwd` | `netdata` | Read-only host view for monitoring (read-only) |
| `/etc/group` | `/host/etc/group` | `netdata` | Read-only host view for monitoring (read-only) |
| `/etc/localtime` | `/etc/localtime` | `netdata` | Host time zone (read-only) |
| `/proc` | `/host/proc` | `netdata` | Read-only host view for monitoring (read-only) |
| `/sys` | `/host/sys` | `netdata` | Read-only host view for monitoring (read-only) |
| `/etc/os-release` | `/host/etc/os-release` | `netdata` | Read-only host view for monitoring (read-only) |
| `/var/log` | `/host/var/log` | `netdata` | Read-only host view for monitoring (read-only) |
| `/var/run/docker.sock` | `/var/run/docker.sock` | `netdata` | Docker API access (grants control of Docker) (read-only) |

## Resource limits

Recommended limits, already set in the compose file under `deploy.resources`. Raise them if the app is starved, or lower them on small hosts.

| Service | CPU limit | Memory limit | Memory reservation | GPU |
|---|---|---|---|---|
| `netdata` | 1.0 | 1000M | - | - |

## Setup

1. Create the folders:
   ```bash
   sudo mkdir -p /opt/stacks/netdata /opt/appdata/netdata
   sudo chown -R $USER: /opt/stacks/netdata
   ```
2. Add the variables above to `/opt/stacks/_shared/.env` and `/opt/stacks/_shared/.secret` (`chmod 600 .secret`).
3. Save the compose file as `/opt/stacks/netdata/compose.yaml`.
4. Check the configuration, then start the stack:
   ```bash
   cd /opt/stacks/netdata
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret config --quiet
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
   ```
5. Watch the first start: `docker compose logs -f`.
6. Open `http://<SERVER_IP>:19999`.
7. Optional: claim the agent to Netdata Cloud for multi-host views, or keep it local only.
8. Review the default alerts under **Alerts**.

> Tip: create a small wrapper such as `alias dc='docker compose --env-file /opt/stacks/_shared/.env --env-file /opt/stacks/_shared/.secret'` so every stack uses the same env files.
{.is-success}

## Updating

```bash
cd /opt/stacks/netdata
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret pull
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
docker image prune -f
```

Back up `/opt/appdata/netdata` before major upgrades. [Watchtower](/watchtower) can automate updates for stateless apps.

## Troubleshooting

- Check status and logs: `docker compose ps` and `docker compose logs --tail=100`.
- **Container names show as IDs:** the Docker socket must be mounted.
- **AppArmor or SELinux denials:** keep `apparmor:unconfined`, or add an SELinux policy on RHEL/Rocky.
- **A variable is empty or reported missing (`:?err`):** run the `config` command above to see which variable is not set in your env files.

## Related

- [Docker Stack Catalog](/docker-stacks)
- [Prometheus](/prometheus)
- [Grafana](/grafana)
- [Dozzle](/dozzle)

---
*Educational example. Review every path, port and credential before you deploy, and never publish your `.secret` file.*
