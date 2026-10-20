---
title: Frigate NVR
description: Frigate NVR 0.18 with OpenVINO CPU object detection and NVIDIA video decode, using Docker Compose.
published: true
date: 2026-10-18T03:28:00.175Z
tags: docker, docker-compose, gpu, homelab, security, cameras, nvr, openvino, photos
editor: markdown
dateCreated: 2026-09-08T21:11:08.018Z
---

# Frigate NVR

> Frigate NVR 0.18 with OpenVINO CPU object detection and NVIDIA video decode, using Docker Compose.
{.is-info}

_Last updated: Oct 2026_

## Overview

Frigate is a network video recorder with real-time AI object detection. It records your IP cameras, detects people, cars and animals, and integrates with Home Assistant. This example uses OpenVINO on the CPU for detection (YOLOv9-tiny ONNX model) and NVIDIA GPUs for ffmpeg decode.

**Why you'd want it:** Local, private camera recording with smart alerts and no cloud subscription.

| Item | Value |
|---|---|
| Stack folder | `/opt/stacks/frigate` |
| Category | Photos and cameras |
| Main image | `ghcr.io/blakeblackshear/frigate:0.18.0` |
| Services | 1 |
| Web UI | https://<SERVER_IP>:8971 (authenticated), http://<SERVER_IP>:30025 (unauthenticated, LAN only) |
| Official docs | https://docs.frigate.video/ |

## Requirements

- A Linux host with Docker Engine and the Docker Compose plugin (see [Docker](/docker))
- The shared folder layout and `.env`/`.secret` files from [Docker Folders](/dockerfolders) and [Environment](/environment)
- IP cameras with RTSP
- Fast storage for recordings
- Optional: NVIDIA GPU with the NVIDIA Container Toolkit for decode
- A modern x86 CPU for OpenVINO detection, or a supported accelerator (Coral, Hailo, etc.)

## Compose

Folder layout:

```text
/opt/stacks/
├── _shared/
│   ├── .env        # shared, non-secret settings
│   └── .secret     # passwords, tokens and keys (chmod 600, never commit)
└── frigate/
    ├── compose.yaml
    └── config/config.yaml
/opt/appdata/frigate/   # persistent application data
```

`/opt/stacks/frigate/compose.yaml`:

```yaml
---
services:
  frigate:
    image: ghcr.io/blakeblackshear/frigate:0.18.0   # Pinned version; exclude from Watchtower and upgrade deliberately
    container_name: frigate
    hostname: frigate
    privileged: true                  # Required for hardware acceleration and some camera features
    env_file:
      - /opt/stacks/_shared/.env
      - /opt/stacks/_shared/.secret

    shm_size: "2048mb"                 # ~2 GB suits roughly 10-12 cameras; raise it if the logs report cache errors

    # FRIGATE_RTSP_PASSWORD comes from _shared/.secret via env_file — used in your config.yaml via {FRIGATE_RTSP_PASSWORD}
    # FRIGATE_GENAI_API_KEY: <API_KEY>   # Uncomment and add your OpenAI key if you enable genai
    logging:
      driver: json-file
      options:
        max-file: ${DOCKERLOGGING_MAXFILE:-10}  # Max number of log files
        max-size: ${DOCKERLOGGING_MAXSIZE:-200k}  # Max file size

    deploy: # NVIDIA GPU passthrough (used for ffmpeg decode)
      resources:
        limits:
          cpus: '4.0'
          memory: 6000M
        reservations:
          memory: 1000M
          devices:
            - driver: nvidia
              count: all  # Use every NVIDIA GPU in the host for video decode
              capabilities: [gpu, video, utility]

    volumes:
      - /opt/stacks/frigate/config:/config                  # Folder containing your config.yaml
      - /opt/appdata/frigate/storage:/media/frigate          # Recordings, snapshots, clips
      - type: tmpfs                       # Fast in-memory cache for recording segments (recommended)
        target: /tmp/cache
        tmpfs:
          size: 1000000000                # 1 GB (increase to 2g+ if you have lots of RAM)

    ports:
      - 30025:5000     # Frigate Web UI No Authentication (root)
      - 8971:8971     # Frigate Web UI (recommended port in newer docs)
      - 8554:8554     # RTSP / go2rtc
      - 8555:8555/tcp # WebRTC over TCP
      - 8555:8555/udp # WebRTC over UDP
        #- 1984:1984     # Uncomment to access go2rtc web interface directly

    restart: unless-stopped
    stop_grace_period: 30s
```

`/opt/stacks/frigate/config/config.yaml`: Minimal example. Add one block per camera.

```yaml
mqtt:
  enabled: false            # set to true and add host/user/password to use Home Assistant

detectors:
  ov:
    type: openvino
    device: CPU             # OpenVINO on the CPU; use GPU for Intel iGPU

model:
  model_type: yolo-generic
  width: 320
  height: 320
  input_tensor: nchw
  input_dtype: float
  path: /config/model_cache/yolov9-t-320.onnx
  labelmap_path: /labelmap/coco-80.txt

ffmpeg:
  hwaccel_args: []          # set per camera below
  input_args: preset-rtsp-generic
  output_args:
    record: preset-record-generic-audio-aac

detect:
  fps: 5
  enabled: true

objects:
  track: [person, car, bicycle, motorcycle, cat, dog]

record:
  enabled: true
  alerts:
    retain:
      days: 30
  detections:
    retain:
      days: 30

snapshots:
  enabled: true
  retain:
    default: 30

go2rtc:
  streams:
    camera_1:
      - rtsp://<CAMERA_USER>:{FRIGATE_RTSP_PASSWORD}@<CAMERA_IP>:554/cam/realmonitor?channel=1&subtype=0   # main stream
    camera_1_sub:
      - rtsp://<CAMERA_USER>:{FRIGATE_RTSP_PASSWORD}@<CAMERA_IP>:554/cam/realmonitor?channel=1&subtype=1   # sub stream

cameras:
  camera_1:
    ffmpeg:
      hwaccel_args: preset-nvidia     # NVIDIA decode; remove for CPU or use preset-vaapi for Intel/AMD
      inputs:
        - path: rtsp://127.0.0.1:8554/camera_1
          roles: [record]
        - path: rtsp://127.0.0.1:8554/camera_1_sub
          roles: [detect]
    detect:
      width: 640
      height: 480
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
| 30025 | 5000 | tcp | `frigate` | Frigate Web UI No Authentication (root) |
| 8971 | 8971 | tcp | `frigate` | Frigate Web UI (recommended port in newer docs) |
| 8554 | 8554 | tcp | `frigate` | RTSP / go2rtc |
| 8555 | 8555 | tcp | `frigate` | WebRTC over TCP |
| 8555 | 8555 | udp | `frigate` | WebRTC over UDP |

## Volumes

| Host path | Container path | Service | Purpose |
|---|---|---|---|
| `/opt/stacks/frigate/config` | `/config` | `frigate` | Folder containing your config.yaml |
| `/opt/appdata/frigate/storage` | `/media/frigate` | `frigate` | Recordings, snapshots, clips |
| tmpfs | `/tmp/cache` | `frigate` | In-memory cache |

## Resource limits

Recommended limits, already set in the compose file under `deploy.resources`. Raise them if the app is starved, or lower them on small hosts.

| Service | CPU limit | Memory limit | Memory reservation | GPU |
|---|---|---|---|---|
| `frigate` | 4.0 | 6000M | 1000M | nvidia x all (gpu, video, utility) |

## Setup

1. Create the folders:
   ```bash
   sudo mkdir -p /opt/stacks/frigate /opt/appdata/frigate
   sudo chown -R $USER: /opt/stacks/frigate
   ```
2. Add the variables above to `/opt/stacks/_shared/.env` and `/opt/stacks/_shared/.secret` (`chmod 600 .secret`).
3. Save the compose file as `/opt/stacks/frigate/compose.yaml` together with the config files shown above.
4. Check the configuration, then start the stack:
   ```bash
   cd /opt/stacks/frigate
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret config --quiet
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
   ```
5. Watch the first start: `docker compose logs -f`.
6. Create `/opt/stacks/frigate/config/config.yaml` from the example below and add your cameras.
7. Download a YOLOv9 ONNX model into `config/model_cache/`, or follow the Frigate docs for the model you choose.
8. Start the stack and open `https://<SERVER_IP>:8971`. The first admin password is printed in `docker logs frigate`.
9. Tune zones, masks and the object list per camera.

> Tip: create a small wrapper such as `alias dc='docker compose --env-file /opt/stacks/_shared/.env --env-file /opt/stacks/_shared/.secret'` so every stack uses the same env files.
{.is-success}

## Updating

```bash
cd /opt/stacks/frigate
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret pull
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
docker image prune -f
```

Some images here are pinned to a version. Read the upstream release notes before changing the tag.

Back up `/opt/appdata/frigate` before major upgrades. [Watchtower](/watchtower) can automate updates for stateless apps.

## Troubleshooting

- Check status and logs: `docker compose ps` and `docker compose logs --tail=100`.
- **`cudaErrorNoKernelImageForDevice`:** older GPUs (e.g. Maxwell) are not supported by the TensorRT image. Use the standard image with OpenVINO or ONNX on the CPU, as shown here.
- **Cache or `shm` errors:** raise `shm_size`, roughly 64 MB per camera plus overhead.
- **Unwanted upgrades:** pin the image tag and exclude `frigate` from Watchtower. Read the release notes before each upgrade.
- Port 30025 (container 5000) has no authentication; keep it off the internet.
- **A variable is empty or reported missing (`:?err`):** run the `config` command above to see which variable is not set in your env files.

## Related

- [Docker Stack Catalog](/docker-stacks)
- [Watchtower](/watchtower)
- [Home Assistant](/home-assistant)

---
*Educational example. Review every path, port and credential before you deploy, and never publish your `.secret` file.*
