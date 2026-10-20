---
title: Graphite Exporter
description: Prometheus Graphite Exporter that accepts Graphite plaintext metrics (e.g. TrueNAS) and exposes them to Prometheus, with Docker Compose.
published: true
date: 2026-10-20T20:01:26.208Z
tags: docker, docker-compose, homelab, metrics, monitoring, prometheus, graphite
editor: markdown
dateCreated: 2026-09-08T21:26:12.838Z
---

# Graphite Exporter

> Prometheus Graphite Exporter that accepts Graphite plaintext metrics (e.g. TrueNAS) and exposes them to Prometheus, with Docker Compose.
{.is-info}

_Last updated: Oct 2026_

## Overview

Many devices and apps (TrueNAS, collectd, some NAS and UPS tools) can only send metrics in Graphite format. Graphite Exporter listens for those metrics on port 9109 and re-publishes them on port 9108 in Prometheus format. A mapping file turns dotted Graphite names into proper Prometheus metric names and labels.

**Why you'd want it:** Bring Graphite-only devices into the same Prometheus and Grafana setup as everything else.

| Item | Value |
|---|---|
| Stack folder | `/opt/stacks/graphite-exporter` |
| Category | Monitoring |
| Main image | `prom/graphite-exporter:latest` |
| Services | 1 |
| Web UI | http://<SERVER_IP>:9108/metrics |
| Official docs | https://github.com/prometheus/graphite_exporter |

## Requirements

- A Linux host with Docker Engine and the Docker Compose plugin (see [Docker](/docker))
- The shared folder layout and `.env`/`.secret` files from [Docker Folders](/dockerfolders) and [Environment](/environment)
- A device or app that can send Graphite plaintext metrics
- [Prometheus](/prometheus) to scrape port 9108

## Compose

Folder layout:

```text
/opt/stacks/
├── _shared/
│   ├── .env        # shared, non-secret settings
│   └── .secret     # passwords, tokens and keys (chmod 600, never commit)
└── graphite-exporter/
    └── compose.yaml
/opt/appdata/graphite-exporter/   # persistent application data
```

`/opt/stacks/graphite-exporter/compose.yaml`:

```yaml
---
services:
  graphite-exporter:
    image: prom/graphite-exporter:latest
    container_name: graphite-exporter
    hostname: graphite-exporter
    env_file:
      - /opt/stacks/_shared/.env
      - /opt/stacks/_shared/.secret
    ports:
      - 9108:9108  # Prometheus scrape endpoint (HTTP)
      - 9109:9109  # Graphite plaintext protocol (TCP)
      - 9109:9109/udp # Graphite plaintext protocol (UDP)
    volumes:
      - /opt/appdata/graphite-exporter/graphite_mapping.conf:/etc/graphite/graphite_mapping.conf:ro
    logging:
      driver: json-file
      options:
        max-file: ${DOCKERLOGGING_MAXFILE:-10}  # Max number of log files
        max-size: ${DOCKERLOGGING_MAXSIZE:-200k}  # Max file size
    command:
      - "--graphite.mapping-config=/etc/graphite/graphite_mapping.conf"
    restart: unless-stopped
```

`/opt/appdata/graphite-exporter/graphite_mapping.conf`:

```yaml
# graphite_exporter mapping rules (statsd_exporter mapping format)
# Example: TrueNAS sends names like servers.<host>.cputemp.temperatures.cpu0
mappings:
  - match: "servers.*.cputemp.temperatures.*"
    name: "truenas_cpu_temperature_celsius"
    labels:
      host: "$1"
      cpu: "$2"
  - match: "servers.*.disktemp.*.temperature"
    name: "truenas_disk_temperature_celsius"
    labels:
      host: "$1"
      disk: "$2"
  # Anything not matched is still exported with a sanitised name.
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
| 9108 | 9108 | tcp | `graphite-exporter` | Prometheus scrape endpoint (HTTP) |
| 9109 | 9109 | tcp | `graphite-exporter` | Graphite plaintext input (TCP and UDP) |
| 9109 | 9109 | udp | `graphite-exporter` | Graphite plaintext input (TCP and UDP) |

## Volumes

| Host path | Container path | Service | Purpose |
|---|---|---|---|
| `/opt/appdata/graphite-exporter/graphite_mapping.conf` | `/etc/graphite/graphite_mapping.conf` | `graphite-exporter` | Metric name mapping rules (read-only) |

## Resource limits

No limits are set. This app is light, so add `deploy.resources.limits` only if you need to cap it.

## Setup

1. Create the folders:
   ```bash
   sudo mkdir -p /opt/stacks/graphite-exporter /opt/appdata/graphite-exporter
   sudo chown -R $USER: /opt/stacks/graphite-exporter
   ```
2. Add the variables above to `/opt/stacks/_shared/.env` and `/opt/stacks/_shared/.secret` (`chmod 600 .secret`).
3. Save the compose file as `/opt/stacks/graphite-exporter/compose.yaml` and create the config file shown above.
4. Create `/opt/appdata/graphite-exporter/graphite_mapping.conf` from the example above. It must exist before the first start, otherwise Docker creates a folder in its place.
5. Check the configuration, then start the stack:
   ```bash
   cd /opt/stacks/graphite-exporter
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret config --quiet
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
   ```
6. Watch the first start: `docker compose logs -f`.
7. Point your device at `<SERVER_IP>` port `9109` as its Graphite server.
8. Check `http://<SERVER_IP>:9108/metrics`, then add a Prometheus scrape job for `<SERVER_IP>:9108`.

> Tip: create a small wrapper such as `alias dc='docker compose --env-file /opt/stacks/_shared/.env --env-file /opt/stacks/_shared/.secret'` so every stack uses the same env files.
{.is-success}

## Updating

```bash
cd /opt/stacks/graphite-exporter
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret pull
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
docker image prune -f
```

Back up `/opt/appdata/graphite-exporter` before major upgrades. [Watchtower](/watchtower) can automate updates for stateless apps.

## Troubleshooting

- Check status and logs: `docker compose ps` and `docker compose logs --tail=100`.
- **Container fails: "is a directory":** the mapping file did not exist on first start. Remove the folder Docker created and create the file.
- **Metrics vanish after 5 minutes:** that is the default `--graphite.sample-expiry`; make sure the device sends regularly.
- **Ugly metric names:** add mapping rules so names and labels are meaningful before you build dashboards.
- **A variable is empty or reported missing (`:?err`):** run the `config` command above to see which variable is not set in your env files.

## Related

- [Docker Stack Catalog](/docker-stacks)
- [Prometheus](/prometheus)
- [Grafana](/grafana)
- [Grafana Alloy](/alloy)

---
*Educational example. Review every path, port and credential before you deploy, and never publish your `.secret` file.*
