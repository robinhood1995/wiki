---
title: Prometheus
description: Prometheus metrics database scraping node exporters, pfSense and Synology SNMP, using Docker Compose.
published: true
date: 2026-10-18T03:28:23.755Z
tags: docker, docker-compose, homelab, metrics, monitoring, prometheus
editor: markdown
dateCreated: 2026-09-08T21:11:08.018Z
---

# Prometheus

> Prometheus metrics database scraping node exporters, pfSense and Synology SNMP, using Docker Compose.
{.is-info}

_Last updated: Oct 2026_

## Overview

Prometheus is a time-series database that scrapes metrics from exporters over HTTP. This example keeps 60 days of data and scrapes node exporters on your Linux hosts, a pfSense firewall and a Synology NAS through the SNMP exporter.

**Why you'd want it:** The standard backend for homelab dashboards and alerting, paired with Grafana.

| Item | Value |
|---|---|
| Stack folder | `/opt/stacks/prometheus` |
| Category | Monitoring |
| Main image | `prom/prometheus:latest` |
| Services | 1 |
| Web UI | http://<SERVER_IP>:9090 |
| Official docs | https://prometheus.io/docs/prometheus/latest/installation/ |

## Requirements

- A Linux host with Docker Engine and the Docker Compose plugin (see [Docker](/docker))
- The shared folder layout and `.env`/`.secret` files from [Docker Folders](/dockerfolders) and [Environment](/environment)
- node_exporter on each host you want to monitor (port 9100)
- Optional: snmp_exporter for NAS/network gear (port 9116)

## Compose

Folder layout:

```text
/opt/stacks/
├── _shared/
│   ├── .env        # shared, non-secret settings
│   └── .secret     # passwords, tokens and keys (chmod 600, never commit)
└── prometheus/
    ├── compose.yaml
    └── prometheus.yml
/opt/appdata/prometheus/   # persistent application data
```

`/opt/stacks/prometheus/compose.yaml`:

```yaml
---
services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    hostname: prometheus
    env_file:
      - /opt/stacks/_shared/.env
      - /opt/stacks/_shared/.secret
    command:
      - '--storage.tsdb.retention.time=60d'
      - '--config.file=/etc/prometheus/prometheus.yml'
    mem_limit: 1g
    cpu_shares: 768
    security_opt:
      - no-new-privileges=true
    user: 0:0
    healthcheck:
      test: wget --no-verbose --tries=1 --spider http://localhost:9090/ || exit 1
    logging:
      driver: json-file
      options:
        max-file: ${DOCKERLOGGING_MAXFILE:-10}  # Max number of log files
        max-size: ${DOCKERLOGGING_MAXSIZE:-200k}  # Max file size
    ports:
    - 9090:9090
    volumes:
      - /opt/appdata/prometheus:/prometheus:rw
      - /opt/stacks/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:rw
    restart: unless-stopped
```

`/opt/stacks/prometheus/prometheus.yml`: Example scrape configuration. Replace the targets with your own hosts.

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'linux_nodes'           # node_exporter on each Linux host / Raspberry Pi
    static_configs:
      - targets: ['<NODE_1_IP>:9100', '<NODE_2_IP>:9100']

  - job_name: 'pfsense'               # pfSense node_exporter package
    static_configs:
      - targets: ['<PFSENSE_IP>:9100']

  - job_name: 'snmp-synology'         # Synology NAS through snmp_exporter
    static_configs:
      - targets: ['<NAS_IP>']
    metrics_path: /snmp
    params:
      module: [synology]
      auth: [snmpv3]
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: <SNMP_EXPORTER_IP>:9116
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
| 9090 | 9090 | tcp | `prometheus` | Web UI |

## Volumes

| Host path | Container path | Service | Purpose |
|---|---|---|---|
| `/opt/appdata/prometheus` | `/prometheus` | `prometheus` | Application data |
| `/opt/stacks/prometheus/prometheus.yml` | `/etc/prometheus/prometheus.yml` | `prometheus` | Persistent data |

## Resource limits

Recommended limits, already set in the compose file under `deploy.resources`. Raise them if the app is starved, or lower them on small hosts.

| Service | CPU limit | Memory limit | Memory reservation | GPU |
|---|---|---|---|---|
| `prometheus` | - | 1g | - | - |

## Setup

1. Create the folders:
   ```bash
   sudo mkdir -p /opt/stacks/prometheus /opt/appdata/prometheus
   sudo chown -R $USER: /opt/stacks/prometheus
   ```
2. Add the variables above to `/opt/stacks/_shared/.env` and `/opt/stacks/_shared/.secret` (`chmod 600 .secret`).
3. Save the compose file as `/opt/stacks/prometheus/compose.yaml` together with the config files shown above.
4. Check the configuration, then start the stack:
   ```bash
   cd /opt/stacks/prometheus
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret config --quiet
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
   ```
5. Watch the first start: `docker compose logs -f`.
6. Save `prometheus.yml` next to `compose.yaml` and list your targets.
7. Start the stack and open **Status > Targets**. Every target should be **UP**.
8. Add Prometheus as a data source in [Grafana](/grafana) (`http://<SERVER_IP>:9090`).
9. To receive metrics from [Grafana Alloy](/alloy), add `--web.enable-remote-write-receiver` to `command`.

> Tip: create a small wrapper such as `alias dc='docker compose --env-file /opt/stacks/_shared/.env --env-file /opt/stacks/_shared/.secret'` so every stack uses the same env files.
{.is-success}

## Updating

```bash
cd /opt/stacks/prometheus
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret pull
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
docker image prune -f
```

Back up `/opt/appdata/prometheus` before major upgrades. [Watchtower](/watchtower) can automate updates for stateless apps.

## Troubleshooting

- Check status and logs: `docker compose ps` and `docker compose logs --tail=100`.
- **Target DOWN:** check the exporter port with `curl http://<NODE_IP>:9100/metrics` from the Prometheus host.
- **Config changes ignored:** restart the container, or enable `--web.enable-lifecycle` and POST to `/-/reload`.
- **Disk filling up:** lower `--storage.tsdb.retention.time`.
- **A variable is empty or reported missing (`:?err`):** run the `config` command above to see which variable is not set in your env files.

## Related

- [Docker Stack Catalog](/docker-stacks)
- [Grafana](/grafana)
- [Grafana Alloy](/alloy)
- [Netdata](/netdata)
- [Graphite Exporter](/graphite-exporter)

---
*Educational example. Review every path, port and credential before you deploy, and never publish your `.secret` file.*
