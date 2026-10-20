---
title: NVIDIA DCGM Exporter
description: NVIDIA DCGM Exporter publishing GPU utilisation, memory, temperature and power metrics to Prometheus, with Docker Compose.
published: true
date: 2026-10-20T20:01:07.619Z
tags: docker, docker-compose, gpu, homelab, monitoring, prometheus, nvidia
editor: markdown
dateCreated: 2026-09-08T21:26:12.838Z
---

# NVIDIA DCGM Exporter

> NVIDIA DCGM Exporter publishing GPU utilisation, memory, temperature and power metrics to Prometheus, with Docker Compose.
{.is-info}

_Last updated: Oct 2026_

## Overview

DCGM Exporter uses NVIDIA Data Center GPU Manager to read per-GPU metrics (utilisation, memory used, temperature, power draw, clocks, encoder/decoder load) and serves them on `/metrics` for Prometheus. Pair it with Grafana for GPU dashboards.

**Why you'd want it:** See how hard Plex, Jellyfin, Frigate or AI workloads push your GPUs, and alert on overheating.

| Item | Value |
|---|---|
| Stack folder | `/opt/stacks/dcgm-exporter` |
| Category | Monitoring |
| Main image | `nvcr.io/nvidia/k8s/dcgm-exporter:2.0.13-2.1.2-ubuntu18.04` |
| Services | 1 |
| Web UI | http://<SERVER_IP>:9400/metrics |
| Official docs | https://github.com/NVIDIA/dcgm-exporter |

## Requirements

- A Linux host with Docker Engine and the Docker Compose plugin (see [Docker](/docker))
- The shared folder layout and `.env`/`.secret` files from [Docker Folders](/dockerfolders) and [Environment](/environment)
- An NVIDIA GPU with the driver installed on the host
- The NVIDIA Container Toolkit (`nvidia-ctk runtime configure --runtime=docker`)
- [Prometheus](/prometheus) to scrape it

## Compose

Folder layout:

```text
/opt/stacks/
├── _shared/
│   ├── .env        # shared, non-secret settings
│   └── .secret     # passwords, tokens and keys (chmod 600, never commit)
└── dcgm-exporter/
    └── compose.yaml
/opt/appdata/dcgm-exporter/   # persistent application data
```

`/opt/stacks/dcgm-exporter/compose.yaml`:

```yaml
---
services:
  dcgm-exporter:
    image: nvcr.io/nvidia/k8s/dcgm-exporter:2.0.13-2.1.2-ubuntu18.04
    container_name: dcgm-exporter
    env_file:
      - /opt/stacks/_shared/.env
      - /opt/stacks/_shared/.secret
    logging:
      driver: json-file
      options:
        max-file: ${DOCKERLOGGING_MAXFILE:-10}  # Max number of log files
        max-size: ${DOCKERLOGGING_MAXSIZE:-200k}  # Max file size
    # Use the NVIDIA runtime
    runtime: nvidia
    # Expose the metrics port on the host
    ports:
      - "9400:9400"
    # Provide access to all GPUs
    deploy:
      resources:
        reservations:
          devices:
            - capabilities: ["gpu"]
    # Optional: mount the procfs for additional system metrics
    volumes:
      - /proc:/host/proc:ro
    # Optional: use the host network namespace
    # network_mode: host
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
| 9400 | 9400 | tcp | `dcgm-exporter` | Web UI |

## Volumes

| Host path | Container path | Service | Purpose |
|---|---|---|---|
| `/proc` | `/host/proc` | `dcgm-exporter` | Read-only host view for monitoring (read-only) |

## Resource limits

Recommended limits, already set in the compose file under `deploy.resources`. Raise them if the app is starved, or lower them on small hosts.

| Service | CPU limit | Memory limit | Memory reservation | GPU |
|---|---|---|---|---|
| `dcgm-exporter` | - | - | - | nvidia (runtime) x all (gpu) |

## Setup

1. Create the folders:
   ```bash
   sudo mkdir -p /opt/stacks/dcgm-exporter /opt/appdata/dcgm-exporter
   sudo chown -R $USER: /opt/stacks/dcgm-exporter
   ```
2. Add the variables above to `/opt/stacks/_shared/.env` and `/opt/stacks/_shared/.secret` (`chmod 600 .secret`).
3. Save the compose file as `/opt/stacks/dcgm-exporter/compose.yaml`.
4. Check the configuration, then start the stack:
   ```bash
   cd /opt/stacks/dcgm-exporter
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret config --quiet
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
   ```
5. Watch the first start: `docker compose logs -f`.
6. Check the endpoint: `curl -s http://<SERVER_IP>:9400/metrics | head`.
7. Add a scrape job in Prometheus: `- job_name: dcgm` with target `<SERVER_IP>:9400`.
8. In [Grafana](/grafana), import the NVIDIA DCGM Exporter dashboard (ID 12239).

> Tip: create a small wrapper such as `alias dc='docker compose --env-file /opt/stacks/_shared/.env --env-file /opt/stacks/_shared/.secret'` so every stack uses the same env files.
{.is-success}

## Updating

```bash
cd /opt/stacks/dcgm-exporter
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret pull
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
docker image prune -f
```

Some images here are pinned to a version. Read the upstream release notes before changing the tag.

Back up `/opt/appdata/dcgm-exporter` before major upgrades. [Watchtower](/watchtower) can automate updates for stateless apps.

## Troubleshooting

- Check status and logs: `docker compose ps` and `docker compose logs --tail=100`.
- **Old image tag:** the tag shown is the one from the source stack. Newer GPUs and drivers need a current tag from NGC (`nvcr.io/nvidia/k8s/dcgm-exporter`); match it to your driver version.
- **`Failed to initialize NVML`:** the NVIDIA runtime is not configured, or the driver is missing on the host.
- **Profiling metrics missing:** some DCGM fields need `cap_add: [SYS_ADMIN]`.
- **Older cards (e.g. Maxwell):** only a subset of fields is reported.
- **A variable is empty or reported missing (`:?err`):** run the `config` command above to see which variable is not set in your env files.

## Related

- [Docker Stack Catalog](/docker-stacks)
- [Prometheus](/prometheus)
- [Grafana](/grafana)
- [nvtop](/nvtop)
- [Netdata](/netdata)

---
*Educational example. Review every path, port and credential before you deploy, and never publish your `.secret` file.*
