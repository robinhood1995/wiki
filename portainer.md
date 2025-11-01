---
title: Portainer
description: 
published: true
date: 2025-10-20T01:58:16.996Z
tags: 
editor: markdown
dateCreated: 2025-10-20T01:58:14.535Z
---

# <img src="/images/portainer.png" width="50" style="vertical-align:middle;margin-right:4px">What's Portainer?

Portainer is a web-based, container management platform that simplifies deploying and managing applications on Docker, Kubernetes, and Podman. It provides an intuitive user interface to perform tasks like managing containers, viewing logs, and adjusting resources without needing to use the command line. Portainer acts as an abstraction layer, making container management easier by hiding the complexity of the underlying platform and allowing teams to work faster and focus on their applications.

## Key features and benefits
### Simplified management: 
Portainer offers a user-friendly interface for managing containers, which allows users to perform actions like deploying, restarting, and debugging applications without being a Kubernetes or Docker expert.
### Unified platform:
It unifies the management of different container environments, including Docker, Kubernetes, and Podman, into a single platform.
### Full visibility:
Users can get a complete overview of their containerized applications across multiple hosts and platforms from a single web page.
### Abstraction of complexity:
It removes the complexity of the underlying container management platform, making it easier for teams to use containers without needing deep expertise. 
### Security and compliance:
Portainer runs exclusively on your own servers, within your network, and does not host any of your infrastructure, which can help with security and compliance needs. 

## How it works
- Portainer's goal is to make container-native application deployment and management accessible by abstracting the complexity of the underlying container management platform.
- It connects to your Docker, Kubernetes, or Podman environments and presents a unified, intuitive UI to manage your applications and infrastructure.
- The tool handles the abstraction for different environments, so users can manage their applications without needing to understand the specifics of each environment.

# Installation
Make sure you visit how to create a folder structure[ docker folders ](/dockerfolders) on how to set them up.
# {.tabset}
## <img src="/images/synology-dsm.png" width="25" style="vertical-align:middle;margin-right:4px">Synology
1. Prepare the environment folder
a. From the the WebUI of Synology using `File Station` navigate to the `docker` folder and then create a folder named `portainer`.

2. Create a YAML file for the portainer configuration
```yaml
###########################################################################
##  Docker Compose File:  Portainer CE (Community Edition)
##  Function:             Docker Management
##  Documentation:        https://hub.docker.com/r/portainer/portainer-ce
###########################################################################
services:
  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer
    restart: unless-stopped
    cpu_shares: 128
    deploy:
      resources:
        limits:
          memory: 128m
        reservations:
          memory: 48m
    security_opt:
      - no-new-privileges:true
    environment:
    - PUID=1024
    - PGID=100
    - TZ=America/Montreal
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /volume1/docker/portainer:/data
    logging:
      driver: json-file
      options:
        max-file: "10"  # Max number of log files
        max-size: "200k"  # Max file size
    ports:
      - 9000:9000
      - 8000:8000 # Edge agent port (if using remote agents)

```
3. Copy the YAML file
a. You will need to create a `compose.yaml` file (we use [VSCode](https://code.visualstudio.com/download)) that and then create the YAML and save it to you local machine.
b. From the the WebUI of Synology using `File Station` navigate to the `docker` folder and then into the `portainer` folder and then upload the `compose.yaml` file.

4. Start up Portainer (older DSM)
a. You will have to login via `ssh` and run the following:
```bash
cd /volume1/docker/portainer
sudo docker compose up -d
```
## <img src="/images/redhat-linux.png" width="25" style="vertical-align:middle;margin-right:4px">RedHat / <img src="/images/rocky-linux.png" width="25" style="vertical-align:middle;margin-right:4px">RedHat Linux
1. Prepare the environment
Create a `portainer` folder under the `docker` folder
```bash
mkdir -p /volume1/docker/portainer
```

2. Create a YAML file for the portainer configuration
```yaml
###########################################################################
##  Docker Compose File:  Portainer CE (Community Edition)
##  Function:             Docker Management
##  Documentation:        https://hub.docker.com/r/portainer/portainer-ce
###########################################################################
services:
  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer
    restart: unless-stopped
    cpu_shares: 128
    deploy:
      resources:
        limits:
            memory: 128m
        reservations:
            memory: 48m
    security_opt:
      - no-new-privileges:true
    environment:
    - PUID=1024
    - PGID=100
    - TZ=America/Montreal
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /volume1/docker/portainer:/data
    logging:
      driver: json-file
      options:
        max-file: "10"  # Max number of log files
        max-size: "200k"  # Max file size
    ports:
      - 9000:9000
      - 8000:8000 # Edge agent port (if using remote agents)

```
3. Copy the YAML file
You will need to cd into the `portainer` folder and then create the YAML file with the above configurations and statr it up from command line:
```bash
cd /volume1/docker/portainer
sudo vi compose.yaml
sudo docker compose up -d
```