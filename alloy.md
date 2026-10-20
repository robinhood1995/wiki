---
title: Grafana Alloy
description: Grafana Alloy collector shipping per-container metrics to Prometheus with Docker Compose.
published: true
date: 2026-10-18T03:27:36.684Z
tags: docker, docker-compose, homelab, grafana, metrics, monitoring, prometheus
editor: markdown
dateCreated: 2026-09-08T21:26:12.838Z
---

# Grafana Alloy

> Grafana Alloy collector shipping per-container metrics to Prometheus with Docker Compose.
{.is-info}

_Last updated: Oct 2026_

## Overview

Grafana Alloy is a telemetry collector. In this stack it runs the built-in cAdvisor exporter to gather CPU, memory, network and disk I/O for every container on the host, then pushes the metrics to Prometheus with `remote_write`. An optional block ships container logs to Loki.

**Why you'd want it:** Per-container resource graphs in Grafana without running cAdvisor separately.

| Item | Value |
|---|---|
| Stack folder | `/opt/stacks/alloy` |
| Category | Monitoring |
| Main image | `grafana/alloy:latest` |
| Services | 1 |
| Web UI | http://<SERVER_IP>:30017 |
| Official docs | https://grafana.com/docs/alloy/latest/ |

## Requirements

- A Linux host with Docker Engine and the Docker Compose plugin (see [Docker](/docker))
- The shared folder layout and `.env`/`.secret` files from [Docker Folders](/dockerfolders) and [Environment](/environment)
- A Prometheus server started with `--web.enable-remote-write-receiver` (see [Prometheus](/prometheus))
- Grafana for dashboards (see [Grafana](/grafana))

## Compose

Folder layout:

```text
/opt/stacks/
├── _shared/
│   ├── .env        # shared, non-secret settings
│   └── .secret     # passwords, tokens and keys (chmod 600, never commit)
└── alloy/
    ├── compose.yaml
    └── config.alloy
/opt/appdata/alloy/   # persistent application data
```

`/opt/stacks/alloy/compose.yaml`:

```yaml
---
services:
  alloy:
    image: grafana/alloy:latest
    container_name: alloy
    hostname: alloy
    env_file:
      - /opt/stacks/_shared/.env
      - /opt/stacks/_shared/.secret
    logging:
      driver: json-file
      options:
        max-file: ${DOCKERLOGGING_MAXFILE:-10}  # Max number of log files
        max-size: ${DOCKERLOGGING_MAXSIZE:-200k}  # Max file size
    command:
      - run
      - --server.http.listen-addr=0.0.0.0:12345
      - --storage.path=/var/lib/alloy/data
      - /etc/alloy/config.alloy
    ports:
      - 30017:12345 # Exposes Alloy's HTTP server
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /opt/stacks/alloy/config.alloy:/etc/alloy/config.alloy:ro
      - /opt/appdata/alloy/data:/var/lib/alloy/data
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 500M
```

`/opt/stacks/alloy/config.alloy`:

```hcl
// config.alloy
// Collects per-container resource metrics (CPU, memory, network, disk I/O)
// from every container on this host via cAdvisor, and forwards them to
// your local Prometheus instance via remote_write.

// --- Metrics: container resource usage -------------------------------

prometheus.exporter.cadvisor "containers" {
  docker_host      = "unix:///var/run/docker.sock"
  storage_duration = "5m"
}

prometheus.scrape "containers" {
  targets         = prometheus.exporter.cadvisor.containers.targets
  forward_to      = [prometheus.remote_write.local.receiver]
  scrape_interval = "15s"
}

prometheus.remote_write "local" {
  endpoint {
    url = "http://<PROMETHEUS_HOST>:9090/api/v1/write"

    // Uncomment if your Prometheus is behind basic auth:
    // basic_auth {
    //   username = "admin"
    //   password = <PASSWORD>
    // }
  }
}

// --- Logs: container stdout/stderr (optional) -------------------------
// Requires a Loki instance running somewhere. Delete this whole section
// if you're not running Loki yet.

// discovery.docker "logs" {
//   host = "unix:///var/run/docker.sock"
// }
//
// discovery.relabel "logs" {
//   targets = discovery.docker.logs.targets
//
//   rule {
//     source_labels = ["__meta_docker_container_name"]
//     regex         = "/(.*)"
//     target_label  = "service_name"
//   }
// }
//
// loki.source.docker "containers" {
//   host             = "unix:///var/run/docker.sock"
//   targets          = discovery.docker.logs.targets
//   relabel_rules    = discovery.relabel.logs.rules
//   forward_to       = [loki.write.local.receiver]
// }
//
// loki.write "local" {
//   endpoint {
//     url = "http://<LOKI_HOST>:3100/loki/api/v1/push"
//   }
// }
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
| 30017 | 12345 | tcp | `alloy` | Exposes Alloy's HTTP server |

## Volumes

| Host path | Container path | Service | Purpose |
|---|---|---|---|
| `/var/run/docker.sock` | `/var/run/docker.sock` | `alloy` | Docker API access (grants control of Docker) (read-only) |
| `/opt/stacks/alloy/config.alloy` | `/etc/alloy/config.alloy` | `alloy` | Persistent data (read-only) |
| `/opt/appdata/alloy/data` | `/var/lib/alloy/data` | `alloy` | Application data |

## Resource limits

Recommended limits, already set in the compose file under `deploy.resources`. Raise them if the app is starved, or lower them on small hosts.

| Service | CPU limit | Memory limit | Memory reservation | GPU |
|---|---|---|---|---|
| `alloy` | 1.0 | 500M | - | - |

## Setup

1. Create the folders:
   ```bash
   sudo mkdir -p /opt/stacks/alloy /opt/appdata/alloy
   sudo chown -R $USER: /opt/stacks/alloy
   ```
2. Add the variables above to `/opt/stacks/_shared/.env` and `/opt/stacks/_shared/.secret` (`chmod 600 .secret`).
3. Save the compose file as `/opt/stacks/alloy/compose.yaml` together with the config files shown above.
4. Check the configuration, then start the stack:
   ```bash
   cd /opt/stacks/alloy
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret config --quiet
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
   ```
5. Watch the first start: `docker compose logs -f`.
6. Save `config.alloy` next to `compose.yaml` and set the Prometheus URL.
7. Start the stack and open `http://<SERVER_IP>:30017` to view the component graph. Every component should be healthy.
8. In Grafana, query `container_cpu_usage_seconds_total` or import a cAdvisor dashboard.

> Tip: create a small wrapper such as `alias dc='docker compose --env-file /opt/stacks/_shared/.env --env-file /opt/stacks/_shared/.secret'` so every stack uses the same env files.
{.is-success}

## Updating

```bash
cd /opt/stacks/alloy
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret pull
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
docker image prune -f
```

Back up `/opt/appdata/alloy` before major upgrades. [Watchtower](/watchtower) can automate updates for stateless apps.

## Troubleshooting

- Check status and logs: `docker compose ps` and `docker compose logs --tail=100`.
- **remote_write 404:** enable the remote write receiver on Prometheus.
- **No containers listed:** the Docker socket must be mounted, and on SELinux hosts you may need `:z` or a policy change.
- **A variable is empty or reported missing (`:?err`):** run the `config` command above to see which variable is not set in your env files.

## Related

- [Docker Stack Catalog](/docker-stacks)
- [Prometheus](/prometheus)
- [Grafana](/grafana)
- [Netdata](/netdata)

---
*Educational example. Review every path, port and credential before you deploy, and never publish your `.secret` file.*
