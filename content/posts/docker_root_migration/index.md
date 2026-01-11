---
title: "Moving Docker's Data directory to another location."
date: "2023-10-30"
tags: ['Computing','Containers','Discussion']
description: "A guide to transferring the Docker data storage to another location while preserving the existing images and containers."
author: ["Naveen Kannan"]
weight: 10
draft: false
ShowToc: true
TocOpen: true
cover:
  image: "docker_logo.jpg"
  alt: "<alt text>"
  caption: "The Docker logo."
  relative: false 
  hidden: false
  hiddenInList: true
  hiddenInSingle: false
params:
  ShowCodeCopyButtons: true
  ShowReadingTime: true
---

# Introduction

Docker is a container service that we have discussed previously in my blog posts.

The default storage location for Docker is at `/var/lib/docker`. Speaking from experience, as images and containers are built over a period of time, especially if there are multiple users using the Docker service, the root filesystem can run into issues where the size of the Docker storage directory can cause potential out-of-space crises, and significantly deteriorate overall system performance. 

In this blog post, we will look at transferring the Storage directory to another location. (In my case, I transferred it to a physical hard drive with expanded storage capacity).  It's an essential step to ensure that Docker operations continue to run smoothly and efficiently, particularly in production environments.

# The process 

The storage transfer process will have the following steps: 

-   Stopping the docker service.

-   Transferring the Docker storage files from `/var/lib/docker` to it's new location, while preserving the file's metadata. 

-   Renaming the old storage so that Docker will not attempt to reuse it. 

-   Creating a config file `daemon.json` at `/etc/docker` in order to point the Docker service to the new storage directory. 

-   Modifying the systemctl service file in order for the Docker service to correctly identify the Docker daemon.**(NOTE: This may or may not be necessary. Explained with greater detail in it's relevant section.)**

-   Reloading the daemons and restarting the Docker service. 

Note that these commands need to be run with elevated (superuser) privileges.

## Stopping the Docker service. 

The Docker service can be stopped with the command:

```bash
systemctl stop docker
```

Systemctl is a Linux command used for controlling and managing system services, including starting, stopping, enabling, and disabling them, among other tasks.

## Transferring the Docker storage files from `/var/lib/docker` to it's new location, while preserving the file's metadata. 

### Creating a new directory for the storage 

```bash
mkdir /path/to/new/storage
```

### Copying the existing storage to the new location

```bash 
rsync -aqxP /var/lib/docker /path/to/new/storage
```

The `rsync` command is used to synchronize and copy files and directories from one location to another. It stands for remote sync, and is a file synchronization tool. 

`-aqxP`:These are the options and flags used with rsync.

-   -a: This option stands for "archive" and is used to perform a recursive copy while preserving file permissions, ownership, timestamps, and more.

-   -q: The "quiet" option suppresses non-error messages, making the output less verbose.

-   -x: This option ensures that rsync does not cross filesystem boundaries. It prevents rsync from copying data to a different filesystem.

-   -P: This combines two options: -P is equivalent to -rlptgD and --partial. It enables resumable copying and shows progress during the transfer.

## Renaming the old Docker storage.

```bash
mv /var/lib/docker /var/lib/docker.old
```

## Creating a `daemon.json` file at `/etc/docker`.

This file will help point the Docker service to the new storage location. 

```bash
echo '{
  "data-root": "/path/to/new/storage"
}' > /etc/docker/daemon.json
```

## Editing the systemctl file to allow Docker to connect to the Docker Daemon

When I attempted to perform the storage transfer on my own, I originally did not have this step included. When restarting the Docker service, I got the following error whenever I attempted any Docker commands. 

```bash
Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?
```
The Docker daemon was very clearly running (by running `systemctl status docker`), but the `docker` cli was unable to connect to the socket file. 

This was because I had restarted the Docker daemon as a service using the `systemctl` command. 

I found my service file path from the output of `systemctl status docker`. 

The following is the first two lines of the output produced by this command. 

```bash
● docker.service - Docker Application Container Engine
   Loaded: loaded (/usr/lib/systemd/system/docker.service; enabled; vendor preset: disabled)
```

Edit this file at the specified location. 

The following was the content in the file on my system. Pay attention to the line that starts with `ExecStart=/usr/bin/dockerd`.

```bash
[Unit]
Description=Docker Application Container Engine
...

[Service]
Type=notify
# the default is not to use systemd for cgroups because the delegate issues still
# exists and systemd currently does not support the cgroup feature set required
# for containers run by docker
ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
...

```

The `-H fd://` means the daemon is using a file descriptor managed by systemctl, without pointing towards the socket file at `/var/run/docker.sock`. 

Modify the line to look like this:

```bash
ExecStart=/usr/bin/dockerd -H unix:///var/run/docker.sock --containerd=/run/containerd/containerd.sock


```
This command can be run to do it as part of a script.

```bash
sed -i 's/^ExecStart=.*/ExecStart=\/usr\/bin\/dockerd -H unix:\/\/\/var\/run\/docker.sock --containerd=\/run\/containerd\/containerd.sock/' /lib/systemd/system/docker.service
```

## Restarting the Daemons and the Docker service

```bash
systemctl daemon-reload
systemctl start docker
```

## Testing!

-   Run `docker image ls` and `docker ps -a` commands to see if the images and containers have been succesfully transferred, and Docker is able to see them at the new location. 

-   Run `docker info` and look for the line that starts with `Docker Root Dir:`. Check if it's pointing towards the new storage location. 

## Deleting the old storage directory

This is a risky step in my opinion. I have the original storage directory preserved at the `/var/lib` folder, only having renamed it to `docker.old`. If you are able to run the Docker service with the new storage location without any issues ( try creating new images and containers!) then it may be a good idea to delete it. (Definitely make a backup of it regardless.)

# Bonus Automation!

The following is a bash script that should automate this process for you while providing a way to check the time taken for each successful step. 

```bash
#!/bin/bash
##################################################
# The purpose of this script is to transfer the Docker storage (root) directory from the root directory (/var/lib/docker) to the hard drive (/path/to/new/storage).

# The following are the steps of this process (in order):

# - Stop the docker service

# - Transfer the Docker storage files from /var/lib/docker to /path/to/new/storage.
        # (Note: This transfer will need to preserve the metadata of the original storage files for Docker to identify it.)

# - Rename the old storage directory so that Docker can no longer identify it.

# - Once the storage has been copied, a new file named daemon.json needs to be placed in /etc/docker to point at the new storage location.

# - Once the storage has been successfully transfered to the new location, the systemctl service file path will need to be modified in order to allow
        # Docker to identify the running Daemon.

# - The daemons need to be reloaded.

# - The Docker service needs to be restarted.

# Note: This script needs to be run as a sudo user.

###################################################

## Stopping the docker service

systemctl stop docker

echo "Docker service stopped at $(date)" >> docker_cache_transfer_progress.txt
echo "." >> docker_cache_transfer_progress.txt
echo "." >> docker_cache_transfer_progress.txt
echo "." >> docker_cache_transfer_progress.txt


## Transferring the Docker storage files from /var/lib/docker to /path/to/new/storage.

## Creating a new file at /path/to/new/storage

mkdir /path/to/new/storage

echo "New storage directory created at /path/to/new/storage . Beginning cache transfer at $(date)." >> docker_cache_transfer_progress.txt

echo "." >> docker_cache_transfer_progress.txt
echo "." >> docker_cache_transfer_progress.txt
echo "." >> docker_cache_transfer_progress.txt

## Copying the Docker storage files

rsync -aqxP /var/lib/docker /path/to/new/storage

echo "Docker storage cloned to /path/to/new/storage at $(date)." >> docker_cache_transfer_progress.txt
echo "." >> docker_cache_transfer_progress.txt
echo "." >> docker_cache_transfer_progress.txt
echo "." >> docker_cache_transfer_progress.txt

## Renaming the old Docker storage

mv /var/lib/docker /var/lib/docker.old

echo "Old Docker storage renamed to /var/lib/docker.old at $(date)." >> docker_cache_transfer_progress.txt
echo "." >> docker_cache_transfer_progress.txt
echo "." >> docker_cache_transfer_progress.txt
echo "." >> docker_cache_transfer_progress.txt

## Creating a daemon.json file at /etc/docker

echo '{
  "data-root": "/path/to/new/storage"
}' > /etc/docker/daemon.json

echo "daemon.json file created at /etc/docker/daemon.json at $(date) with the following content:" >> docker_cache_transfer_progress.txt

cat /etc/docker/daemon.json >> docker_cache_transfer_progress.txt
echo "." >> docker_cache_transfer_progress.txt
echo "." >> docker_cache_transfer_progress.txt
echo "." >> docker_cache_transfer_progress.txt

## Editing the systemctl file
# The following command will replace the line beginning with ExecStart with the appropriate line needed to actually have the systemctl comman connect to the Docker daemon.

sed -i 's/^ExecStart=.*/ExecStart=\/usr\/bin\/dockerd -H unix:\/\/\/var\/run\/docker.sock --containerd=\/run\/containerd\/containerd.sock/' /lib/systemd/system/docker.servi$

echo "systemctl config file modified at $(date)." >> docker_cache_transfer_progress.txt
echo "." >> docker_cache_transfer_progress.txt
echo "." >> docker_cache_transfer_progress.txt
echo "." >> docker_cache_transfer_progress.txt

## Restarting the daemons

systemctl daemon-reload

echo "daemons reloaded at $(date)" >> docker_cache_transfer_progress.txt
echo "." >> docker_cache_transfer_progress.txt
echo "." >> docker_cache_transfer_progress.txt
echo "." >> docker_cache_transfer_progress.txt

## Restarting the Docker service

systemctl start docker

echo "Docker service started at $(date)" >> docker_cache_transfer_progress.txt

```