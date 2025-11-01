---
title: Docker Folders
description: 
published: true
date: 2025-10-20T01:16:30.210Z
tags: 
editor: markdown
dateCreated: 2025-10-19T23:58:33.872Z
---

# <img src="/images/foldergear.png" width="50" style="vertical-align:middle;margin-right:4px">Folder Structure
Creating Shared Folders for the Docker environments.
> 📌 For this we will be using the folowing `folders`:
>```
>docker
>data
>```
>💡NOTE: These folders are the defaults we will be using throughout the site.
{.is-info}

Folder structure example, this applies to all enviroments to aide the moveability in-between flavors of OS:
```bash
/volume1/docker
/volume1/data
```
# {.tabset}
## <img src="/images/synology-dsm.png" width="25" style="vertical-align:middle;margin-right:4px">Synology

1. Create Shared Folders:
a. Open Control Panel > Shared Folder.
b. Click Create > Create Shared Folder.
c. As you are creating make sure that the "Admin" user has read/write access to this folder
d. Add a `docker` share and a `data` share

2. Notes
a. The `docker` folder will house your Docker applications and the `data` will house all of your media or other data.
b. This will add all of the directories under the `admin` user and the `users` group.

>NOTE: If you are going to create a SERVARR stack then you can create another shared folder, for example, named media or data, to store your actual media files (movies, TV shows, music, etc.) that your Arr applications will manage.

3. Create Subfolders within the Shared Folders:
a. Open File Station on your Synology NAS.
b. Navigate to the docker shared folder.
c. On another note, you will have to create any folders by had via the `File Station` application within synology for any Docker persistant volume.

>Create a subfolder named appdata. Inside appdata, create individual subfolders for each Arr application you plan to use (e.g., radarr, sonarr, lidarr, prowlarr, etc.). These will store the configuration files for each application.
Navigate to the media (or data) shared folder.
Create subfolders within this folder to organize your media, such as movies, tv, music.

## <img src="/images/redhat-linux.png" width="25" style="vertical-align:middle;margin-right:4px">RedHat / <img src="/images/rocky-linux.png" width="25" style="vertical-align:middle;margin-right:4px">Rocky Linux
1. To create the specified folder structure for Docker volumes on a Linux system, follow these steps: 
a. Create the base directories.
b. Create the `/volume1` directory
c. Create the `/data` directory

These will serve as the host directories for your Docker volumes.
```bash
sudo mkdir -p /volume1/docker
sudo mkdir -p /volume1/data
```

2. Set the users
a. Create a `admin` user to match Synology
b. Create a `docker` user to match Synology
```bash
sudo useradd -m admin -g users
sudo useradd -m docker -g users
```
3. Set the file permissions (we use admin for all)
```bash
sudo chmod -R a=,a+rX,u+w,g+w /volume1/docker
sudo chown -R admin:users /volume1/docker
sudo chmod -R a=,a+rX,u+w,g+w /volume1/data
sudo chown -R admin:users /volume1/data
```
4. Get the id information from the users (needed for later)
```bash
id admin
id docker
```
This should being back the following information on each user
```
uid=1024(admin) gid=100(users) groups=100(users)
uid=1031(docker) gid=100(users) groups=100(users)
```
User id 1024 and/or 1031 and 100 for the group
