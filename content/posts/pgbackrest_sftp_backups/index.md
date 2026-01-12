---
title: "Secure SFTP Backups with pgBackRest for PostgreSQL: A Step-by-Step Guide"
date: "2025-05-14"
tags: ['PostgreSQL', 'pgBackRest', 'Database Backups', 'Linux Administration', 'DevOps', 'Security']
description: "A guide to configuring pgBackRest to securely perform SFTP based backups of your PostgreSQL server."
author: ["Naveen Kannan"]
weight: 10
draft: false
ShowToc: true
TocOpen: true
params:
  ShowCodeCopyButtons: true
  ShowReadingTime: true
---

# Introduction

pgBackRest is an open source tool that allows you to perform automated backup and restore operations for a PostgreSQL server. It allows you to take both full and incremental file system backups of your data. With pgBackRest, you can set up multiple repositories for your backups, both local and remote. You can further customize the configuration to take backups at specific times of the day so that regular operations are not affected, and backup retention and rotation can be customized as needed.

Furthermore, it allows mirroring of backups to cloud-compatible object stores, including S3, Azure, and GCS compatible object stores.

{{< figure
    src="pgbackrest_logo.png"
    caption="pgBackRest helps make taking database backups pretty straightforward."
    align=center
>}}

Insecurely configured SFTP backups can expose critical database content to unauthorized access, yet many organizations rely on them for off-site storage. This guide addresses the security gap and will explain the process of setting up remote SFTP repositories in a secure manner. We will create a system user on the SFTP repository and limit their permissions, and perform scheduled backups exclusively via the limited user account.

A good database backup strategy uses multiple repositories of different types, alongside both physical and logical backups. Do not limit yourself to merely SFTP repositories. Remember, with database backups, redundancy is good. This tutorial is simply to explain how to *securely* configure SFTP backups.

## Database Server Requirements

-   PostgreSQL 12.0 or newer (this tutorial uses PostgreSQL 17.0)

    -   PostgreSQL 12.0+ is required because earlier versions lack WAL archiving features that enable point-in-time recovery when using pgBackRest.

-   pgBackRest 2.41 or newer (this tutorial uses pgBackRest 2.55.1)

    -   Earlier versions may have limitations with certain authentication methods

-   OpenSSH 8.0+ client for improved Ed25519 key support

-   libssh2 1.9.0+ for modern cryptographic algorithm support

-   Sufficient disk space for temporary backup files before transfer

## SFTP Host Requirements

-   Linux server with SSH daemon supporting SFTP subsystem

    -   Recommended: OpenSSH 8.0+ for optimal security features

-   Sufficient storage space for your backup retention needs. Plan for minimum 3x your database size (1x for full backup + 1x for transaction logs + 1x overhead and retention)

-   Root access for initial configuration

-   Ability to create dedicated system users and groups

-   Firewall configured to allow SFTP connections (typically port 22) from your database server IP

## Network Requirements

-   Stable network connection between database and SFTP servers

-   Firewall rules allowing SFTP traffic (TCP port 22 by default)

-   Recommended: Internal network connectivity (avoid public internet exposure if possible)

You can check your versions with these commands:

``` bash
postgres --version
pgbackrest version
ssh -V
```

# The Process

For this process, this guide assumes both servers are on a private network (10.0.0.0/24) protected by a corporate firewall, with the following IP addresses:

-   The IP of the database server is `10.0.0.1`

-   The user that runs the code on the database server is `postgres`. `postgres` is a system user and cannot log in via SSH.

-   The IP of the SFTP repo is `10.0.0.2`

## On the database host

-   Create an SSH key pair in the `postgres` user's home directory.

```bash
# Adjust this path based on your system's postgres home
export PGHOME=/var/lib/postgresql
ssh-keygen -f $PGHOME/.ssh/id_ed25519_sftp_key -t ed25519 -N ""

```

Ensure that the `.ssh` folder is owned by the `postgres` user. We use Ed25519 keys because they provide stronger security with shorter keys compared to RSA, and are supported by modern OpenSSH implementations.

-   Perform a key scan on the SFTP host to gather the public keys. Make sure you verify the legitimacy of the host server before you build your known hosts file.

```bash
ssh-keyscan 10.0.0.2 >> $PGHOME/.ssh/known_hosts
```

You can validate the host keys via fingerprints by running:

```bash
ssh-keygen -lf $PGHOME/.ssh/known_hosts
```

## On the SFTP server host

### *Create a directory that the SFTP user account will be restricted to.*

We create a dedicated /data directory separate from standard system paths to isolate backup data and simplify permission management.

```bash
# Creating a folder in the root directory
mkdir /data

# Creating the home directory for the SFTP user
mkdir -p /data/sftpuser/upload

## Setting the appropriate permissions for this filesystem
chmod 701 /data

## /data needs to have root:root permissions
chown root:root /data
```

### *Create an SFTP group, followed by an SFTP user with minimal permissions and as a system user.*

```bash
# Creating an SFTP group
groupadd sftpusers
# Creating the SFTP user as a system user
useradd -g sftpusers -d /data/sftpuser/upload/ -s /usr/sbin/nologin -r sftpuser
```

This process creates a group called `sftpusers` and creates a system user in that group called `sftpuser`, with their home directory being `/data/sftpuser/upload`.

### *Change the permissions for the SFTP directory.*

```bash
## Giving ownership to the sftpusers group for the sftpuser folder
chown root:sftpusers /data/sftpuser

## Giving the sftpuser system account access to the home directory we created
chown sftpuser:sftpusers /data/sftpuser/upload
```

It is very important that `/data/sftpuser/upload` must *not* be owned by root. This is a common cause of failure for chroot setups.

### *Updating the SSH configuration on the SFTP server.*

Add the following to the end `/etc/ssh/sshd_config`:

```bash
# For sftp backups
Match Group sftpusers
  ChrootDirectory /data/%u
  ForceCommand internal-sftp
  AllowTcpForwarding no
  X11Forwarding no
```

Here's what this does:

-   The first line `Match Group sftpusers` starts a conditional block that only applies to users in the `sftpusers` group.

-   The second line `ChrootDirectory /data/%u` (jails) the user to the directory `/data/username`, where `%u` is replaced with the `username`. This ensures the user is jailed to their upload directory and cannot access the broader file system.

-   The third line `ForceCommand internal-sftp` forces the SSH session to use the internal SFTP subsystem. The user cannot get a shell session, and only SFTP is allowed.

-   The fourth line `AllowTcpForwarding no` prevents the user using SSH port forwarding to avoid tunneling through SSH.

-   The fifth line `X11Forwarding no` disables X11 forwarding to prevent GUI application tunneling over SSH.

### *Validate your SSH config and then Restart the SSH daemon*

```bash
sudo sshd -T && sudo systemctl restart sshd
```

::: callout-warning
Do not restart the SSH daemon without validating your SSH config with `sudo sshd -T`! If your config is wrong, the daemon will fail to initiate, and you may be locked out of being able to SSH into the server!
:::

### *Create the SSH directory for the sftpuser account with appropriate permissions*

```bash
# Creating the .ssh directory
mkdir /data/sftpuser/upload/.ssh

## Creating the authorized_keys and known_hosts files
touch /data/sftpuser/upload/.ssh/authorized_keys
touch /data/sftpuser/upload/.ssh/known_hosts

# Changing permissions for the newly created folders and files
chown -R sftpuser:sftpusers /data/sftpuser/upload/.ssh
chmod 700 /data/sftpuser/upload/.ssh
chmod 640 /data/sftpuser/upload/.ssh/authorized_keys
chmod 644 /data/sftpuser/upload/.ssh/known_hosts
```

### *Append the contents of the new public SSH key to the authorized_keys file*

On the database host, copy the contents of the public key we created (`$PGHOME/.ssh/id_ed25519_sftp_key.pub`) and append it to the authorized_keys file on the SFTP user's SSH directory ('/data/sftpuser/upload/.ssh/authorized_keys').

::: callout-warning
Remember to *never share your private key* with any server, or to anyone else! Only copy the contents of the public key!
:::

# Testing

From the database host, test if you can login to the SFTP server using the keys without needing a password.

```bash
sftp -i $PGHOME/.ssh/id_ed25519_sftp_key sftpuser@10.0.0.2
```

If your setup was correct, this will let you login without a password.

# Deploying

Update your `pgbackrest.config` file with the following:

```ini
# SFTP Backup Repo
repo2-bundle=y
repo2-path=/upload
# Host
repo2-sftp-host=10.0.0.2
repo2-sftp-host-key-hash-type=sha256
# User
repo2-sftp-host-user=sftpuser
# Key Location
repo2-sftp-private-key-file=/var/lib/postgresql/.ssh/id_ed25519_sftp_key
repo2-sftp-public-key-file=/var/lib/postgresql/.ssh/id_ed25519_sftp_key.pub
repo2-type=sftp
```

The above configuration file sets up the automation process for transferring backups to the SFTP host via the SFTP service, as the user `sftpuser`, who has restricted permissions.

An example of a full `pgbackrest.conf` file: (The default location is `/etc/pgbackrest/pgbackrest.conf`)

```ini
[main]
pg1-path=/usr/local/pgsql/data
pg1-port=5432
pg1-user=postgres
pg1-database=postgres

[global]
# Local Backup Repo
repo1-cipher-pass=xxx # Insert with your own
repo1-cipher-type=aes-256-cbc
repo1-path=/var/lib/pgbackrest
repo1-retention-full=2

# SFTP Backup Repo 1
# This section handles the configuration for your SFTP repo
repo2-bundle=y
repo2-path=/upload
repo2-retention-full=2
# Host
repo2-sftp-host=10.0.0.2
repo2-sftp-host-key-hash-type=sha256
# User
repo2-sftp-host-user=sftpuser
# Key Location
repo2-sftp-private-key-file=/var/lib/postgresql/.ssh/id_ed25519_sftp_key
repo2-sftp-public-key-file=/var/lib/postgresql/.ssh/id_ed25519_sftp_key.pub

# Logging Info
log-path=/var/log/pgbackrest
log-level-console=debug
[global:archive-push]
compress-level=3
```

Then create the stanza by running the following as the `postgres` user:

```bash
pgbackrest --stanza=main --log-level-console=info stanza-create
```

If your config is right, then this will succeed with the following output:

```bash
P00   INFO: stanza-create command end: completed successfully
```

# Addendum

## Using secure key algorithms and hash functions

If you are using older versions of cryptography libraries like libssh2 and openssl, your configuration might be restricted to using older, more outdated key algorithms like RSA or DSS, along with older hash functions like SHA1. I recommend using more secure, newer key algorithms like Ed25519, along with more secure hashing functions like SHA256.

### Check the key algorithms supported by your installation of libssh2

Run the following code to see what key types are supported by your installation of libssh2:

```bash
strings /usr/lib/x86_64-linux-gnu/libssh2.so.1 | grep ssh-
```

Your output should return more modern key algorithms like `ssh-ed25519`. If you output only includes algorithms like `ssh-rsa` or `ssh-dss`, then you will need to update your installation of libssh2. Some older servers, like ones running outdated operating systems like Ubuntu 20.04, may not allow you to update your libssh2 library using package managers. In that case, you will have to manually build the newer versions from source.

```bash
## Installing dependencies
sudo apt update
sudo apt install build-essential cmake zlib1g-dev libssl-dev

# Downloading and building libssh2 from source
wget https://www.libssh2.org/download/libssh2-1.11.0.tar.gz
tar -xzf libssh2-1.11.0.tar.gz
cd libssh2-1.11.0

# Installing libssh2
./configure
make -j$(nproc)
sudo make install
sudo ldconfig
```

The new libssh2 library will have it's binaries located at `/usr/local/lib/libssh2.so`. You can confirm if it was installed by running:

```bash
strings /usr/local/lib/libssh2.so | grep ssh-
```

You should see the newer key algorithms like `ecdsa-sha2-nistp256`and `ssh-ed25519`.

If you built/installed pgBackRest *before* updating your libssh2 library, you will need to rebuild it from source so that it uses the newer library.

To check if pgBackRest is using the newly installed libssh2 as a dependency, run:

```bash
ldd $(which pgbackrest) | grep libssh2
```

Your output should look like this:

```bash
libssh2.so.1 => /usr/local/lib/libssh2.so.1 (0x00007f20882cf000)
```

This confirms that pgBackRest is using the manually installed libssh2 from `/usr/local/lib`, rather than the system-wide default at `/usr/lib/x86_64-linux-gnu`.

# References

-   [pgBackRest documentation](https://pgbackrest.org/user-guide.html)

-   [libssh2](https://libssh2.org/)

-   [PostgreSQL backup documentation](https://www.postgresql.org/docs/current/backup.html)
