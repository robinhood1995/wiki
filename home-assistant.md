---
title: Home Assistant
description: 
published: true
date: 2025-12-30T18:10:29.146Z
tags: 
editor: markdown
dateCreated: 2025-12-30T18:10:29.146Z
---

# <img src="/images/home-assistant-alt.png" width="50" style="vertical-align:middle;margin-right:4px">Home Assistant Introduction
Home Assistant is an extremely popular open-source home automation platform that puts local control and privacy first.
It allows you to connect and control almost all your smart home devices (lights, plugs, sensors, cameras, thermostats, locks, blinds, media players, etc.) from a single, unified interface — without relying on vendor clouds whenever possible.

# Installation
1. The first step is to make sure you are up to date.
```bash
    sudo dnf update -y
```
2. Install dnf-plugins-core: This package provides utilities to manage DNF repositories.
```bash
    sudo dnf install -y dnf-plugins-core
```
# Useful Commands
- Reset Password: (you must access the console)
	Execute the command: `ha auth reset --username your_username --password your_new_password`

# {.tabset}
## <img src="/images/redhat-linux.png" width="25" class="tab-icon"> RedHat Linux

To install Docker Engine on Red Hat Enterprise Linux (RHEL) and start the service, follow these steps:

3. Uninstall older versions (if any):
```bash
sudo dnf remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-selinux \
                  docker-engine-selinux \
                  docker-engine
```
4. Add the Docker repository: This ensures you install the latest Docker Community Edition (CE) packages.
```bash
sudo dnf -y install dnf-utils
sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
````
5. Install Docker Engine: Install the necessary Docker packages, including the engine, CLI, containerd runtime, and buildx/compose plugins.
```bash
sudo dnf install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
```
6. Start and enable the Docker service: This will ensure Docker starts automatically on boot.
```bash
sudo systemctl start docker
sudo systemctl enable docker
```
7. (Optional) Add your user to the docker group: This allows you to run Docker commands without sudo. Replace $(whoami) with a specific username if adding another user.
```bash
sudo usermod -aG docker $USER
newgrp docker
```
After this step, you will need to log out and log back in for the changes to take effect.
8. Verify the installation: Run a test container to confirm Docker is working correctly.
```bash
docker run hello-world
```
## <img src="/images/rocky-linux.png" width="25" class="tab-icon"> Rocky Linux
To install Docker Engine on Rocky Linux and start the service, follow these steps:

3. Uninstall older versions (if any):
```bash
sudo dnf remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-selinux \
                  docker-engine-selinux \
                  docker-engine
```
4. Add the Docker repository: This ensures you install the latest Docker Community Edition (CE) packages.
```bash
sudo dnf -y install dnf-utils
sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
````
5. Install Docker Engine: Install the necessary Docker packages, including the engine, CLI, containerd runtime, and buildx/compose plugins.
```bash
sudo dnf install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
```
6. Start and enable the Docker service: This will ensure Docker starts automatically on boot.
```bash
sudo systemctl start docker
sudo systemctl enable docker
```
7. (Optional) Add your user to the docker group: This allows you to run Docker commands without sudo. Replace $(whoami) with a specific username if adding another user.
```bash
sudo usermod -aG docker $USER
newgrp docker
```
After this step, you will need to log out and log back in for the changes to take effect.
8. Verify the installation: Run a test container to confirm Docker is working correctly.
```bash
docker run hello-world
```

